---
read_when:
    - Her Codex harness yapılandırma alanına ihtiyacınız var
    - Uygulama sunucusunun aktarım, kimlik doğrulama, keşif veya zaman aşımı davranışını değiştiriyorsunuz
    - Codex harness başlatma, model keşfi veya ortam yalıtımında hata ayıklıyorsunuz
summary: Codex harness'ı için yapılandırma, kimlik doğrulama, keşif ve uygulama sunucusu başvurusu
title: Codex çalıştırma ortamı referansı
x-i18n:
    generated_at: "2026-07-27T00:06:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 149f065f5bef18d0f491c97facc4b5991afc3f7e1077abdc7a4b49f506eac3e0
    source_path: plugins/codex-harness-reference.md
    workflow: 16
---

Bu referans, resmi `codex` Plugin'i için ayrıntılı yapılandırmayı kapsar.
Kurulum ve yönlendirme kararları için
[Codex harness](/tr/plugins/codex-harness) ile başlayın.

## Plugin yapılandırma yüzeyi

Tüm Codex harness ayarları `plugins.entries.codex.config` altında bulunur.

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          discovery: {
            enabled: true,
            timeoutMs: 2500,
          },
          appServer: {
            mode: "guardian",
          },
        },
      },
    },
  },
}
```

Üst düzey alanlar:

| Alan                       | Varsayılan               | Anlamı                                                                                                                                         |
| -------------------------- | ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `discovery`                | etkin                    | Codex app-server `model/list` için model keşfi ayarları.                                                                                        |
| `appServer`                | yönetilen stdio app-server | Aktarım, komut, kimlik doğrulama, onay, sanal alan ve zaman aşımı ayarları. Standart harness varsayılan olarak agent kapsamlı durumu kullanır.  |
| `codexDynamicToolsLoading` | `"searchable"`           | OpenClaw dinamik araçlarını doğrudan başlangıçtaki Codex araç bağlamına yerleştirmek için `"direct"` kullanın.                                 |
| `codexDynamicToolsExclude` | `[]`                     | Codex app-server turlarında hariç tutulacak ek OpenClaw dinamik araç adları.                                                                    |
| `codexPlugins`             | devre dışı               | Bağlı hesap uygulamalarına isteğe bağlı erişim dâhil olmak üzere yerel Codex Plugin/uygulama desteği. Bkz. [Yerel Codex Plugin'leri](/tr/plugins/codex-native-plugins). |
| `computerUse`              | devre dışı               | Codex Computer Use kurulumu. Bkz. [Codex Computer Use](/tr/plugins/codex-computer-use).                                                            |
| `sessionCatalog`           | etkin                    | Kenar çubuğu için yerel Codex oturum keşfi. Sağlayıcıyı veya harness'ı devre dışı bırakmadan keşfi devre dışı bırakmak için `enabled: false` ayarlayın. |
| `supervision`              | devre dışı               | Agent'a yönelik yerel oturum transkripti ve yazma denetimi politikası. Bkz. [Codex gözetimi](/plugins/codex-supervision).                       |

## Gözetim

Yerel oturum keşfi, varsayılan olarak Gateway bilgisayarındaki ve katılımı etkinleştirilmiş eşleştirilmiş Node'lardaki arşivlenmemiş Codex oturumlarını listeler. Yalnızca bu kataloğu şu şekilde devre dışı bırakın:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          sessionCatalog: {
            enabled: false,
          },
        },
      },
    },
  },
}
```

`supervision`, agent'a yönelik araçları ayrıca denetler:

| Alan                  | Varsayılan              | Anlamı                                                                                                                                                                                                                                    |
| --------------------- | ----------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `enabled`             | `false`                 | Agent'a yönelik Codex gözetim araçlarını etkinleştirir. Bu, kimliği doğrulanmış operatör oturum kataloğunu denetlemez.                                                                                                                      |
| `endpoints`           | yerleşik yerel uç nokta | Korunan Codex gözetim agent'ı ve bağımsız MCP araçları için uyumluluk ve gelişmiş uç nokta hedefleri. İnsan kataloğu ve dal akışı bu hedefleri yok sayar ve `appServer` üzerinden çözümlenen gözetim App Server'ını kullanır.             |
| `allowRawTranscripts` | `false`                 | Gözetim etkinken özerk agent veya bağımsız MCP transkript okumalarına ve transkriptten türetilen liste alanlarına izin verir. Yalnızca `codex_threads` meta verilerini okuma özelliği kullanılabilir durumda kalır. Kimliği doğrulanmış Control UI devamını denetlemez. |
| `allowWriteControls`  | `false`                 | Gözetim etkinken özerk `codex_threads` çatallama, yeniden adlandırma, arşivleme ve arşivden çıkarma değişikliklerinin yanı sıra bağımsız MCP gönderme, yönlendirme ve kesme işlemlerine izin verir. Diğer bağlama, ana makine, durum veya onay kontrollerini atlamaz. |

Uç nokta girdileri şu alanları kabul eder:

| Alan           | Geçerli olduğu yer | Anlamı                                                                |
| -------------- | ------------------ | --------------------------------------------------------------------- |
| `id`           | tümü               | Kararlı uç nokta kimliği.                                             |
| `label`        | tümü               | İsteğe bağlı görüntüleme etiketi.                                     |
| `transport`    | tümü               | `"stdio-proxy"` veya `"websocket"`.                                   |
| `command`      | `stdio-proxy`    | İsteğe bağlı App Server komutu.                                       |
| `args`         | `stdio-proxy`    | İsteğe bağlı komut bağımsız değişkenleri.                              |
| `cwd`          | `stdio-proxy`    | İsteğe bağlı alt işlem çalışma dizini.                                 |
| `url`          | `websocket`      | Gerekli WebSocket veya desteklenen yerel soket URL'si.                 |
| `authTokenEnv` | `websocket`      | Değeri uç noktanın kimliğini doğrulayan isteğe bağlı ortam değişkeni.  |

**Codex Sessions** sayfası, Plugin'in gözetim App Server'ını kullanır ve yalnızca arşivlenmemiş oturumları gösterir. Açık `appServer` bağlantı ayarları olmadığında bu bağlantı, yönetilen kullanıcı ana dizini stdio'sudur. Saklanan veya boşta olan yerel satırlar, son kalıcı terminal kaynak turuna kadar sınırlı kullanıcı ve asistan geçmişiyle modele kilitli bir Sohbet oluşturabilir. Özel bağlaması; anlık görüntü çatalını, kurallı `appServer` kaynak dalını, geçmiş eklemeyi ve sonraki turları bu bağlantıda tutar. İlk kurallı başlatma, çatalın döndürdüğü çifti kullanır. Daha sonraki sürdürmelerde OpenClaw model ve sağlayıcı geçersiz kılmaları atlanır; böylece Codex, kurallı iş parçacığının kalıcı çiftini geri yükler. Ayrı bir yerel değişiklik bu çifti güncelleyebilir, ancak dış model ve geri dönüş zinciri hiçbir zaman onun yerini almaz. Saklanan ve boşta olan satırlar, başka bir çalıştırıcı olmadığı onaylandıktan sonra arşivlenebilir; ancak başka bir etkin OpenClaw bağlaması tam hedefin veya onun arşivlenmemiş oluşturulmuş alt öğelerinden birinin sahibiyse bu işlem yapılamaz. OpenClaw, Codex'in alt öğe sayfalandırmasını izler ve numaralandırma hatalarında, döngülerde veya güvenlik sınırının tükenmesinde güvenli biçimde başarısız olur. Onay, bilinmeyen yerel istemcileri ve durumdan arşivlemeye geçiş yarışını kapsamaya devam eder. Gözetimli, modele kilitli bir Sohbet, yerel bağlamayı korurken silinemez. Etkin kaynaklar bir dal oluşturamaz veya arşivlenemez, ancak mevcut bir gözetimli Sohbet yine de açılabilir. Eşleştirilmiş Node satırlarının tümü salt okunur kalır; Node aktarımı henüz harness'ın ihtiyaç duyduğu akış yaşam döngüsünü sağlamaz.

Yalnızca `appServer.homeScope: "user"`, yönetilen bir harness işleminin hangi Codex ana dizinini kullandığını değiştirir; filo kataloğunu yayımlamaz. Gözetimi etkinleştirmek harness varsayılanını değiştirmez. Bunun yerine, açık `appServer` bağlantı ayarları yoksa ayrı gözetim bağlantısı varsayılan olarak yönetilen kullanıcı ana dizini stdio'sunu kullanır. Açık ayarlar bu bağlantı için dikkate alınır. Bekleyen ve kaydedilmiş gözetimli bağlamalar bu bağlantıyı her turda korur; devre dışı gözetim veya bağlantı/yaşam döngüsü sapması, agent ana dizini harness'ına geri dönmek yerine güvenli biçimde başarısız olur. Varsayılan bağlantı, yerel Codex istemcileriyle saklanan oturumları paylaşır; istemcilerin işlem yerelindeki etkinlik durumunu paylaşmaz.

Eski `plugins.entries.codex-supervisor` ayarları kullanımdan kaldırılmıştır. Eski girdiyi, uç nokta tanımlarını, politika bayraklarını ve Plugin izin/verme-reddetme başvurularını bu bloğa taşımak için
`openclaw doctor --fix` komutunu çalıştırın. Çakışmalarda açık kurallı
`codex.config.supervision` değerleri önceliklidir.

## App-server aktarımı

Standart harness turlarında OpenClaw, resmi Plugin ile gönderilen yönetilen Codex ikili dosyasını (şu anda `@openai/codex` `0.145.0`) başlatır:

```bash
codex app-server --listen stdio://
```

Bu, app-server sürümünü yerel olarak yüklü olabilecek ayrı bir Codex CLI yerine resmi `codex` Plugin'ine bağlı tutar. Yalnızca kasıtlı olarak farklı bir yürütülebilir dosya kullanmak istediğinizde `appServer.command` ayarlayın. Varsayılan yalıtılmış agent ana dizinine sahip standart yönetilen turlar, bir macOS masaüstü paketi yüklü olsa bile bu sabitlenmiş paketi tercih eder. [Computer Use](/tr/plugins/codex-computer-use) etkinken veya `homeScope`, `"user"` olduğunda ve yerel Computer Use durumunu yükleyebildiğinde yönetilen başlatma bunun yerine gerekli macOS izinlerinin sahibi olan masaüstü uygulaması ikili dosyasını tercih eder. Aynı masaüstü öncelikli kuralı, yalıtılmış bir agent ana dizininin etkin Codex yapılandırması yerel Computer Use'ı etkinleştirdiğinde de geçerlidir. Yüklü bir masaüstü uygulaması paketi yoksa OpenClaw, sabitlenmiş paket ikili dosyasına geri döner.

Yürütülebilir dosya devri ve yerel yapılandırma sınırlaması, çalışan tek bir Gateway işlemi içindeki istemcileri koordine eder. Başka bir işlem yerel Codex Plugin yapılandırmasını değiştirdikten sonra Gateway'i yeniden başlatın.

Gözetim ayrı bir bağlantıyı çözümler. Açık
`appServer` bağlantı ayarları olmadığında, `homeScope: "user"` ile yönetilen stdio'yu kullanır; standart harness ise `homeScope: "agent"` ile yönetilen stdio olarak kalır. Açık bağlantı ayarları her iki yol tarafından da dikkate alınır. Standart harness'ın `$CODEX_HOME` (veya `~/.codex`) öğesini yerel istemcilerle paylaşması gerektiğinde `homeScope: "user"` değerini açıkça ayarlayın. Özel bir gözetimli bağlama, standart harness varsayılanından bağımsız olarak gözetim bağlantısını kullanır. Bağımsız App Server işlemleri ayrı canlı durum ve onay durumlarını korur.

Hâlihazırda çalışan bir app-server'a karşı üretim dışı testler için WebSocket aktarımı kullanılabilir:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            transport: "websocket",
            url: "ws://gateway-host:39175",
            authToken: "${CODEX_APP_SERVER_TOKEN}",
            requestTimeoutMs: 60000,
          },
        },
      },
    },
  },
}
```

Codex, WebSocket aktarımını deneysel ve desteklenmeyen olarak sınıflandırır. Üretim iş yükleri için yönetilen stdio'yu veya yerel Unix denetim soketini tercih edin.

`appServer` alanları:

| Alan                                          | Varsayılan                                             | Anlamı                                                                                                                                                                                                                                                                                                                                                                                          |
| --------------------------------------------- | ------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `transport`                                   | `"stdio"`                                              | `"stdio"` Codex'i başlatır; açıkça belirtilen `"unix"` yerel denetim soketine bağlanır; `"websocket"`, `url` hedefine bağlanır.                                                                                                                                                                                                                                                       |
| `homeScope`                                   | `"agent"`                                              | `"agent"`, sıradan çalıştırma sistemi durumunu her OpenClaw aracısı için yalıtır. `"user"`, yerel `$CODEX_HOME` veya `~/.codex` öğesini paylaşan, yerel kimlik doğrulamayı kullanan ve yalnızca sahip tarafından iş parçacığı yönetimini etkinleştiren açık bir katılım seçeneğidir. Kullanıcı kapsamı yerel stdio veya Unix aktarımını destekler. Ayrı denetim bağlantısı için ayarlanmamış bir değer, stdio ya da Unix için `"user"`, WebSocket için ise `"agent"` olarak çözümlenir. |
| `command`                                     | yönetilen Codex ikili dosyası                          | stdio aktarımının yürütülebilir dosyası. Yönetilen ikili dosyayı kullanmak için ayarlamadan bırakın.                                                                                                                                                                                                                                                                                             |
| `args`                                        | `["app-server", "--listen", "stdio://"]`               | stdio aktarımının bağımsız değişkenleri.                                                                                                                                                                                                                                                                                                                                                        |
| `url`                                         | ayarlanmamış                                           | WebSocket App Server URL'si veya `unix://` URL'si. Açıkça belirtilen boş bir Unix yolu, standart kullanıcı ana dizini denetim soketini seçer.                                                                                                                                                                                                                                            |
| `authToken`                                   | ayarlanmamış                                           | WebSocket aktarımının Bearer belirteci. Değişmez bir dizeyi veya `${CODEX_APP_SERVER_TOKEN}` gibi bir SecretInput değerini kabul eder.                                                                                                                                                                                                                                                                    |
| `headers`                                     | `{}`                                                   | Ek WebSocket üstbilgileri. Üstbilgi değerleri değişmez dizeleri veya SecretInput değerlerini kabul eder; örneğin `x-codex-client-session-token: "${CODEX_CLIENT_SESSION_TOKEN}"`.                                                                                                                                                                                                                                                             |
| `clearEnv`                                    | `[]`                                                   | OpenClaw devralınan ortamını oluşturduktan sonra, başlatılan stdio app-server işleminden kaldırılacak ek ortam değişkeni adları.                                                                                                                                                                                                                                                                 |
| `remoteWorkspaceRoot`                         | ayarlanmamış                                           | Uzak Codex app-server çalışma alanı kökü. Ayarlandığında OpenClaw, çözümlenen OpenClaw çalışma alanından yerel çalışma alanı kökünü çıkarır, geçerli cwd son ekini bu uzak kök altında korur ve Codex'e yalnızca nihai app-server cwd değerini gönderir. cwd, çözümlenen OpenClaw çalışma alanı kökünün dışındaysa OpenClaw, uzak app-server'a Gateway'e yerel bir yol göndermek yerine güvenli biçimde başarısız olur. |
| `loopDetectionPreToolUseRelay`                | `true`                                                 | Yalnızca OpenClaw döngü algılaması ve bunun açık politikasızlık işaretçisi için kullanılan Codex `PreToolUse` alt işlemini kurar. Araç başına işlem dallanmasını azaltmak için `false` olarak ayarlayın. Araç öncesi Plugin kancaları ve güvenilir araç politikası, gerekli aktarıcılarını yine de kurar.                                                                                       |
| `requestTimeoutMs`                            | `60000`                                                | app-server denetim düzlemi çağrılarının zaman aşımı.                                                                                                                                                                                                                                                                                                                                            |
| `turnCompletionIdleTimeoutMs`                 | `60000`                                                | OpenClaw `turn/completed` beklerken, Codex'in bir turu kabul etmesinden veya tur kapsamlı bir app-server isteğinden sonraki sessiz pencere.                                                                                                                                                                                                                                                     |
| `turnAssistantCompletionIdleTimeoutMs`        | `10000`                                                | OpenClaw hâlâ `turn/completed` beklerken, son/açıklama niteliğinde olmayan bir asistan öğesinin veya araç öncesi ham asistan tamamlamasının asistan çıktısı serbest bırakımını devreye almasından sonraki sessiz pencere. Bu değeri artırmak, OpenClaw kesinti yapıp oturum hattını serbest bırakmadan önce Codex'e `turn/completed` yayımlaması için daha fazla zaman tanır.                          |
| `postToolRawAssistantCompletionIdleTimeoutMs` | `300000`                                               | OpenClaw `turn/completed` beklerken; bir araç devri, yerel araç tamamlanması, araç sonrası ham asistan ilerlemesi, ham muhakeme tamamlanması veya muhakeme ilerlemesinden sonra kullanılan tamamlanma-boşta kalma ve ilerleme koruması. Araç sonrası sentezin son asistan serbest bırakma bütçesinden meşru olarak daha uzun süre sessiz kalabildiği güvenilir veya ağır iş yükleri için bunu kullanın. |
| `mode`                                        | yerel Codex gereksinimleri YOLO'ya izin vermediği sürece `"yolo"` | YOLO veya koruyucu tarafından incelenen yürütme için ön ayar.                                                                                                                                                                                                                                                                                                                                   |
| `approvalPolicy`                              | `"never"` veya izin verilen bir koruyucu onay politikası | İş parçacığı başlatma, sürdürme ve tura gönderilen yerel Codex onay politikası.                                                                                                                                                                                                                                                                                                                  |
| `sandbox`                                     | `"danger-full-access"` veya izin verilen bir koruyucu korumalı alanı | İş parçacığı başlatma ve sürdürmeye gönderilen yerel Codex korumalı alan modu. Etkin OpenClaw korumalı alanları, `danger-full-access` turlarını Codex `workspace-write` ile sınırlar; turun ağ bayrağı OpenClaw korumalı alan çıkışını izler.                                                                                                                                                           |
| `approvalsReviewer`                           | `"user"` veya izin verilen bir koruyucu inceleyici      | İzin verildiğinde Codex'in yerel onay istemlerini incelemesini sağlamak için `"auto_review"` kullanın.                                                                                                                                                                                                                                                                                         |
| `defaultWorkspaceDir`                         | geçerli işlem dizini                                   | `--cwd` belirtilmediğinde `/codex bind` tarafından kullanılan çalışma alanı.                                                                                                                                                                                                                                                                                                    |
| `serviceTier`                                 | ayarlanmamış                                           | İsteğe bağlı Codex app-server hizmet katmanı. `"priority"` hızlı mod yönlendirmesini etkinleştirir, `"flex"` esnek işlemeyi talep eder ve `null` geçersiz kılmayı temizler. Eski `"fast"`, `"priority"` olarak kabul edilir.                                                                                                                                |
| `networkProxy`                                | devre dışı                                             | app-server komutları için Codex izin profili ağ kullanımına katılım sağlar. OpenClaw, seçilen `permissions.<profile>.network` yapılandırmasını tanımlar ve `sandbox` göndermek yerine bunu `default_permissions` ile seçer.                                                                                                                                                                                |
| `experimental.sandboxExecServer`              | `false`                                                | Yerel Codex yürütmesinin etkin OpenClaw korumalı alanı içinde çalışabilmesi için OpenClaw korumalı alanı destekli bir Codex ortamını desteklenen Codex uygulama sunucusuna kaydeden önizleme katılım seçeneği.                                                                                                                                                                                                            |

`appServer.networkProxy`, Codex sandbox sözleşmesini değiştirdiği için açıkça belirtilir. Etkinleştirildiğinde OpenClaw, oluşturulan izin profilinin Codex tarafından yönetilen ağ iletişimini başlatabilmesi için Codex iş parçacığı yapılandırmasında `features.network_proxy.enabled` ve
`default_permissions` değerlerini de ayarlar. OpenClaw, varsayılan olarak profil gövdesinden çakışmaya dayanıklı bir `openclaw-network-<fingerprint>` profil adı oluşturur; `profileName` seçeneğini yalnızca kararlı bir yerel ad gerektiğinde kullanın.

```js
export default {
  plugins: {
    entries: {
      codex: {
        config: {
          appServer: {
            sandbox: "workspace-write",
            networkProxy: {
              enabled: true,
              domains: {
                "api.openai.com": "allow",
                "blocked.example.com": "deny",
              },
              allowUpstreamProxy: true,
              proxyUrl: "http://127.0.0.1:3128",
            },
          },
        },
      },
    },
  },
};
```

Normal uygulama sunucusu çalışma zamanı `danger-full-access` olacaksa `networkProxy` seçeneğinin etkinleştirilmesi, oluşturulan izin profili için bunun yerine çalışma alanı tarzı dosya sistemi erişimini kullanır. Codex tarafından yönetilen ağ uygulaması sandbox içinde ağ iletişimi olduğundan, tam erişimli bir profil giden trafiği korumaz.

Plugin; eski, daha yeni fakat doğrulanmamış, ön sürüm, derleme son eki içeren veya sürümsüz uygulama sunucusu el sıkışmalarını engeller. Codex uygulama sunucusu, `0.143.0` ile paketlenmiş `0.145.0` arasındaki kararlı bir sürümü bildirmelidir.

OpenClaw, geri döngü dışındaki WebSocket uygulama sunucusu URL'lerini uzak olarak değerlendirir ve `appServer.authToken` ya da bir `Authorization` üst bilgisi aracılığıyla kimlik taşıyan WebSocket kimlik doğrulaması gerektirir. `appServer.authToken` ve her `appServer.headers.*` değeri bir SecretInput olabilir; gizli bilgiler çalışma zamanı, OpenClaw uygulama sunucusu başlatma seçeneklerini oluşturmadan önce SecretRef'leri ve env kısaltmasını çözümler; çözümlenmemiş yapılandırılmış SecretRef'ler ise herhangi bir token veya üst bilgi gönderilmeden önce başarısız olur. Yerel Codex plugin'leri yapılandırıldığında OpenClaw, bu plugin'leri yüklemek veya yenilemek için bağlı uygulama sunucusunun plugin denetim düzlemini kullanır ve ardından plugin'lerin sahip olduğu uygulamaların Codex iş parçacığında görünür olması için uygulama envanterini yeniler. `app/list` hâlâ yetkili envanter ve meta veri kaynağıdır; ancak Codex söz konusu uygulamayı şu anda devre dışı olarak işaretlese bile OpenClaw ilkesi, listelenmiş ve erişilebilir bir uygulama için `thread/start` öğesinin `config.apps[appId].enabled = true` gönderip göndermeyeceğine karar verir. Bilinmeyen veya eksik uygulama kimlikleri kapalı kalır; bu yol yalnızca `plugin/install` aracılığıyla pazar yeri plugin'lerini etkinleştirir ve envanteri yeniler. OpenClaw'ı yalnızca OpenClaw tarafından yönetilen plugin yüklemelerini ve uygulama envanteri yenilemelerini kabul edeceğine güvenilen uzak uygulama sunucularına bağlayın.

## Onay ve sandbox modları

Yerel stdio uygulama sunucusu oturumları varsayılan olarak YOLO modunu kullanır:
`approvalPolicy: "never"`, `approvalsReviewer: "user"` ve
`sandbox: "danger-full-access"`. Bu güvenilir yerel operatör yaklaşımı, yanıtlayacak kimse bulunmadığında gözetimsiz OpenClaw turlarının ve Heartbeat'lerin yerel onay istemleri olmadan ilerlemesini sağlar.

Codex'in yerel sistem gereksinimleri dosyası örtük YOLO onayı, inceleyici veya sandbox değerlerine izin vermiyorsa OpenClaw, örtük varsayılanı bunun yerine guardian olarak değerlendirir ve izin verilen guardian izinlerini seçer. `tools.exec.mode: "auto"` ayrıca guardian tarafından incelenen Codex onaylarını zorunlu kılar ve güvenli olmayan eski `approvalPolicy: "never"` veya `sandbox: "danger-full-access"` geçersiz kılmalarını korumaz; kasıtlı olarak onaysız bir yaklaşım için `tools.exec.mode: "full"` ayarını yapın. Aynı gereksinimler dosyasındaki ana bilgisayar adıyla eşleşen `[[remote_sandbox_config]]` girdileri, sandbox varsayılanı kararında dikkate alınır.

Guardian tarafından incelenen Codex onayları için `appServer.mode: "guardian"` ayarını yapın:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            mode: "guardian",
            serviceTier: "priority",
          },
        },
      },
    },
  },
}
```

Bu değerlere izin verildiğinde `guardian` ön ayarı `approvalPolicy: "on-request"`,
`approvalsReviewer: "auto_review"` ve `sandbox: "workspace-write"` olarak genişler. Tek tek ilke alanları `mode` değerini geçersiz kılar. Eski `guardian_subagent` inceleyici değeri hâlâ uyumluluk diğer adı olarak kabul edilir; ancak yeni yapılandırmalarda `auto_review` kullanılmalıdır.

Bir OpenClaw sandbox'ı etkinken yerel Codex uygulama sunucusu işlemi yine Gateway ana bilgisayarında çalışır. Bu nedenle OpenClaw, Codex'in ana bilgisayar tarafı sandbox kullanımını OpenClaw sandbox arka ucuyla eşdeğer saymak yerine söz konusu tur için Codex'in yerel Code Mode özelliğini, kullanıcı MCP sunucularını ve uygulama destekli plugin yürütmesini devre dışı bırakır. Normal exec/process araçları kullanılabilir olduğunda kabuk erişimi, `sandbox_exec` ve `sandbox_process` gibi OpenClaw sandbox destekli dinamik araçlar üzerinden sunulur.

<Note>
Docker destekli OpenClaw sandbox ana bilgisayarlarında (`agents.defaults.sandbox.mode` bir Docker arka ucuna ayarlandığında) `openclaw doctor`, ana bilgisayarın ayrıcalıksız kullanıcı ad alanlarına ve Docker sandbox ağ çıkışı devre dışı bırakıldığında ağ ad alanlarına izin verip vermediğini yoklar; sandbox kapsayıcısındaki `workspace-write` kabuk yürütmesi için iç içe Codex `bwrap` bunlara ihtiyaç duyar. Başarısız bir yoklama, Ubuntu/AppArmor ana bilgisayarlarında genellikle `bwrap: setting up uid map: Permission denied` veya
`bwrap: loopback: Failed RTM_NEWADDR: Operation not permitted` olarak ortaya çıkar. OpenClaw hizmet kullanıcısı için bildirilen ana bilgisayar ad alanı ilkesini düzeltin ve Gateway'i yeniden başlatın; ana bilgisayar genelindeki `kernel.apparmor_restrict_unprivileged_userns=0` geri dönüşü yerine hizmet işlemi için kapsamlı bir AppArmor profilini tercih edin ve yalnızca iç içe `bwrap` gereksinimini karşılamak amacıyla daha geniş Docker kapsayıcı ayrıcalıkları vermeyin.
</Note>

## Sandbox içindeki yerel yürütme

Kararlı varsayılan davranış kapalı başarısızlıktır: etkin OpenClaw sandbox kullanımı, aksi hâlde Codex uygulama sunucusu ana bilgisayarından çalışacak yerel Codex yürütme yüzeylerini devre dışı bırakır. `appServer.experimental.sandboxExecServer: true` seçeneğini yalnızca Codex'in uzak ortam desteğini OpenClaw'ın sandbox arka ucuyla denemek istediğinizde kullanın. Bu önizleme yolu, desteklenen her Codex uygulama sunucusu sürümüyle çalışır.

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            experimental: {
              sandboxExecServer: true,
            },
          },
        },
      },
    },
  },
}
```

Bayrak açıkken ve geçerli OpenClaw oturumu sandbox içindeyken OpenClaw, etkin sandbox tarafından desteklenen yerel bir geri döngü exec sunucusu başlatır, bunu Codex uygulama sunucusuna kaydeder ve Codex iş parçacığı ile turunu OpenClaw'a ait bu ortamla başlatır. Uygulama sunucusu ortamı kaydedemezse çalıştırma, sessizce ana bilgisayar yürütmesine geri dönmek yerine kapalı olarak başarısız olur.

Bu önizleme yolu yalnızca yereldir. Uzak bir WebSocket uygulama sunucusu aynı ana bilgisayarda çalışmadığı sürece geri döngü exec sunucusuna erişemez; bu nedenle OpenClaw bu birleşimi reddeder.

## Kimlik doğrulama ve ortam yalıtımı

Varsayılan aracı başına ana dizinde kimlik doğrulama şu sırayla seçilir:

1. Aracı için açıkça belirtilmiş bir OpenClaw Codex kimlik doğrulama profili.
2. Uygulama sunucusunun söz konusu aracının Codex ana dizinindeki mevcut hesabı.
3. Yalnızca yerel stdio uygulama sunucusu başlatmalarında, hiçbir uygulama sunucusu hesabı yoksa ve OpenAI kimlik doğrulaması hâlâ gerekliyse önce `CODEX_API_KEY`, ardından
   `OPENAI_API_KEY`.

OpenClaw, ChatGPT aboneliği tarzında bir Codex kimlik doğrulama profili (OAuth veya token kimlik bilgisi türü) gördüğünde, oluşturulan Codex alt işleminden `CODEX_API_KEY` ve `OPENAI_API_KEY` değerlerini kaldırır. Bu, Gateway düzeyindeki API anahtarlarını gömmeler veya doğrudan OpenAI modelleri için kullanılabilir tutarken yerel Codex uygulama sunucusu turlarının yanlışlıkla API üzerinden ücretlendirilmesini önler.

Açık Codex API anahtarı profilleri ve yerel stdio env anahtarı geri dönüşü, devralınan alt işlem env'si yerine uygulama sunucusu oturum açma işlemini kullanır. WebSocket uygulama sunucusu bağlantıları Gateway env API anahtarı geri dönüşünü almaz; açık bir kimlik doğrulama profili veya uzak uygulama sunucusunun kendi hesabını kullanın.

Stdio uygulama sunucusu başlatmaları varsayılan olarak OpenClaw'ın işlem ortamını devralır. OpenClaw, Codex uygulama sunucusu hesap köprüsünün sahibidir ve `CODEX_HOME` değerini söz konusu aracının OpenClaw durumu altındaki aracı başına bir dizine ayarlar. Bu, Codex yapılandırmasını, hesaplarını, plugin önbelleğini/verilerini ve iş parçacığı durumunu operatörün kişisel `~/.codex` ana dizininden sızdırmak yerine OpenClaw aracısı kapsamında tutar.

Yerel Codex durumunu Codex Desktop ve CLI ile paylaşmak için `appServer.homeScope: "user"` ayarını yapın. Bu yerel kullanıcı ana dizini modu, yönetilen stdio ve açık Unix aktarımını destekler. Ayarlanmışsa `$CODEX_HOME`, aksi hâlde `~/.codex` kullanır; buna yerel kimlik doğrulama, yapılandırma, plugin'ler ve iş parçacıkları dahildir. OpenClaw, uygulama sunucusu için kimlik doğrulama profili köprüsünü atlar. Doğrulanmış sahip turları, bu iş parçacıklarını listelemek (isteğe bağlı bir `search` filtresiyle), okumak, çatallamak, yeniden adlandırmak, arşivlemek ve arşivden çıkarmak için `codex_threads` kullanabilir. Bir iş parçacığını OpenClaw'da sürdürmeden önce çatallayın; bağımsız Codex işlemleri aynı iş parçacığındaki eşzamanlı yazıcıları koordine etmez.

Bu `homeScope` katılımı sıradan harness oturumları için geçerlidir. Codex Sessions aracılığıyla oluşturulan bir Chat ise bunun yerine özel gözetim bağlantısını kullanır; bu bağlantı, standart dal ve gelecekteki sürdürmeler için yerel bağlantının kimlik doğrulama ve sağlayıcı yapılandırmasını korur.

Modeli kilitlenmiş gözetimli bir Chat'te `codex_threads`, farklı bir çatalı bağlayamaz veya Chat'e bağlı yerel iş parçacığını arşivleyemez. Listeleme ve yalnızca meta veri okuma kullanılabilir kalır. Ham transkript okumaları `allowRawTranscripts` gerektirir; bu devre dışı bırakıldığında yerel arama transkript önizlemeleriyle eşleşebileceğinden liste araması da reddedilir. Başka bir OpenClaw Chat'ine ait olmayan ilgisiz bir iş parçacığını yeniden adlandırmak, arşivden çıkarmak, ayrık olarak çatallamak ve arşivlemek için `allowWriteControls` gerekir. Her iki seçenek de kilitli bir bağlamayı atlamaz.

OpenClaw, normal yerel uygulama sunucusu başlatmaları için `HOME` değerini yeniden yazmaz. `openclaw`, `gh`, `git`, bulut CLI'ları ve kabuk komutları gibi Codex tarafından çalıştırılan alt işlemler normal işlem ana dizinini görür ve kullanıcı ana dizini yapılandırmasını ve token'larını bulabilir. Codex ayrıca `$HOME/.agents/skills` ve
`$HOME/.agents/plugins/marketplace.json` öğelerini keşfedebilir; bu `.agents` keşfi kasıtlı olarak operatör ana diziniyle paylaşılır ve yalıtılmış `~/.codex` durumundan ayrıdır.

Varsayılan aracı kapsamında OpenClaw plugin'leri ve OpenClaw skill anlık görüntüleri yine OpenClaw'ın kendi plugin kayıt defteri ve skill yükleyicisi üzerinden akar; kişisel Codex `~/.codex` varlıkları akmaz. Yalıtılmış bir OpenClaw aracısının parçası olması gereken, bir Codex ana dizininden gelen kullanışlı Codex CLI skill'leriniz veya plugin'leriniz varsa bunların envanterini açıkça çıkarın:

```bash
openclaw migrate codex --dry-run
openclaw migrate apply codex --yes
```

Bir dağıtım ek ortam yalıtımına ihtiyaç duyuyorsa bu değişkenleri `appServer.clearEnv` öğesine ekleyin:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            clearEnv: ["CODEX_API_KEY", "OPENAI_API_KEY"],
          },
        },
      },
    },
  },
}
```

`appServer.clearEnv` yalnızca oluşturulan Codex uygulama sunucusu alt işlemini etkiler. OpenClaw, yerel başlatma normalleştirmesi sırasında `CODEX_HOME` ve `HOME` öğelerini bu listeden kaldırır: `CODEX_HOME` seçilen aracı veya kullanıcı kapsamını göstermeyi sürdürür ve alt işlemlerin normal kullanıcı ana dizini durumunu kullanabilmesi için `HOME` devralınmaya devam eder.

## Dinamik araçlar

Codex dinamik araçları varsayılan olarak `searchable` yüklemesini kullanır ve `openclaw` ad alanı altında `deferLoading: true` ile sunulur. OpenClaw normalde Codex'in yerel çalışma alanı işlemlerini veya Codex'in kendi araç arama yüzeyini yineleyen dinamik araçları sunmaz:

- `read`
- `write`
- `edit`
- `apply_patch`
- `exec`
- `process`
- `update_plan`
- `tool_call`
- `tool_describe`
- `tool_search`
- `tool_search_code`

Sonlu bir çalışma zamanı izin listesi yerel Code Mode'u devre dışı bıraktığında OpenClaw boş bir yürütme ortamı seçimi gönderir. Bu doğrudan, sandbox'sız durumda OpenClaw, ilkeye göre filtrelenmiş `exec` ve `process` araçlarını kabuk geri dönüşü olarak tutar. Çalışma zamanı izin listeleri ve `codexDynamicToolsExclude` uygulanmaya devam eder.

OpenClaw entegrasyon araçlarının mesajlaşma, medya, cron, tarayıcı, Node'lar, Gateway, `heartbeat_respond` ve `web_search` gibi geri kalan çoğu, bu ad alanı altındaki Codex araç araması aracılığıyla kullanılabilir. Bu, başlangıçtaki model bağlamını daha küçük tutar. Codex araç araması kullanılamayabileceği veya yalnızca bağlayıcılardan oluşan bir evreni çözümleyebileceği için küçük bir araç kümesi, `codexDynamicToolsLoading` değerinden bağımsız olarak doğrudan çağrılabilir durumda kalır: `agents_list`, `sessions_spawn` ve `sessions_yield`. Geliştirici talimatları, Codex'e özgü alt ajan çalışmaları için normal Codex alt ajanlarını yerel `spawn_agent` yönünde yönlendirmeye devam ederken `sessions_spawn`, açık OpenClaw veya ACP delegasyonu için kullanılabilir durumda kalır. Yalnızca mesaj aracı kaynaklı yanıtlar da doğrudan kalır; çünkü bu, bir tur denetimi sözleşmesidir.

Codex Code Mode, genel OpenClaw dinamik araç sonuçlarını metin olarak sunar. Alanları okumadan önce bir JSON sonucunu ayrıştırın. İç içe dinamik çağrılar Codex çalışma zamanı tarafından seri hâle getirilir; bu nedenle `Promise.all` bunları eşzamanlı olarak göndermez. Toplayıcı alt süreçlerini başlatırken sınırlı bir sıralı başlatma döngüsü kullanın.

OpenClaw `computer` aracı dâhil olmak üzere `catalogMode: "direct-only"` olarak işaretlenen araçlar, `openclaw_direct` altında gruplandırılır. OpenClaw, operatör tarafından sağlanan girdileri değiştirmeden bu ad alanını Codex'in `code_mode.direct_only_tool_namespaces` listesine ekler. Bu nedenle Codex, bu araçları iç içe Code Mode `tools.*` çağrıları üzerinden yönlendirmek yerine normal ve yalnızca kod modu iş parçacıklarında `DirectModelOnly` olarak sunar. Bu sınır, görüntü içeren sonuçlar için gereklidir: İç içe Code Mode serileştirmesi görüntü çıktısını metne dönüştürerek bir sonraki bilgisayar eylemi için gereken ekran görüntüsünü yok eder.

`codexDynamicToolsLoading: "direct"` değerini yalnızca ertelenmiş dinamik araçlarda arama yapamayan özel bir Codex uygulama sunucusuna bağlanırken veya tam araç yükünde hata ayıklarken ayarlayın.

## Zaman aşımları

OpenClaw'a ait dinamik araç çağrıları, `appServer.requestTimeoutMs` değerinden bağımsız olarak sınırlandırılır. Her Codex `item/tool/call` isteği, aşağıdaki sırayla kullanılabilir ilk zaman aşımını kullanır:

- Çağrı başına pozitif bir `timeoutMs` bağımsız değişkeni.
- `image_generate` için `agents.defaults.mediaModels.image.timeoutMs`.
- Yapılandırılmış bir zaman aşımı bulunmayan `image_generate` için 120 saniyelik varsayılan görüntü oluşturma süresi.
- Medya anlama `image` aracı için seçilen görüntü özellikli `tools.media.models[]` girdisinin milisaniyeye dönüştürülmüş `timeoutSeconds` değeri veya 60 saniyelik varsayılan medya süresi. Görüntü anlamada bu süre isteğin kendisine uygulanır ve önceki hazırlık çalışmaları nedeniyle azaltılmaz.
- `message` aracı için Gateway teslimatını ve aynı anahtara yönelik sınırlı uzlaştırmayı kapsayan sabit 600 saniyelik dış bütçe.
- 90 saniyelik varsayılan dinamik araç süresi.

Bu gözetleyici, dış dinamik `item/tool/call` bütçesidir. Sağlayıcıya özgü istek zaman aşımları bu çağrı içinde çalışır ve kendi zaman aşımı semantiklerini korur. Dinamik araç bütçeleri 600000 ms ile sınırlandırılır. `agents_wait`, 30000 ms dış tamamlama ek süresi ekler ve yapılandırılmış bekleme sonucunun Codex'e ulaşabilmesi için uygulama sunucusu istemcisi 660000 ms süre tanır. Zaman aşımında OpenClaw, desteklendiği durumlarda araç sinyalini iptal eder ve Codex'e başarısız bir dinamik araç yanıtı döndürür; böylece oturum `processing` durumunda bırakılmadan tur devam edebilir.

Codex bir turu kabul ettikten ve OpenClaw tur kapsamındaki bir uygulama sunucusu isteğine yanıt verdikten sonra koşum takımı, Codex'in geçerli turda ilerlemesini ve sonunda yerel turu `turn/completed` ile tamamlamasını bekler. Uygulama sunucusu `appServer.turnCompletionIdleTimeoutMs` boyunca sessiz kalırsa OpenClaw, mümkün olan en iyi şekilde Codex turunu keser, tanılama amaçlı bir zaman aşımı kaydeder ve sonraki sohbet mesajlarının eski bir yerel turun arkasında kuyruğa alınmaması için OpenClaw oturum hattını serbest bırakır.

Aynı tura ait terminal olmayan bildirimlerin çoğu bu kısa gözetleyiciyi devre dışı bırakır; çünkü Codex, turun hâlâ etkin olduğunu kanıtlamıştır. Araç devirleri daha uzun bir araç sonrası boşta kalma bütçesi kullanır: OpenClaw bir `item/tool/call` yanıtı döndürdükten, `commandExecution` gibi yerel araç öğeleri tamamlandıktan, ham `custom_tool_call_output` tamamlamalarından ve araç sonrası ham asistan ilerlemesi, ham akıl yürütme tamamlamaları veya akıl yürütme ilerlemesinden sonra. Koruma, yapılandırıldığında `appServer.postToolRawAssistantCompletionIdleTimeoutMs` değerini kullanır; aksi takdirde varsayılan olarak beş dakika kullanır. Aynı araç sonrası bütçe, Codex bir sonraki geçerli tur olayını yayımlamadan önceki sessiz sentez penceresi için ilerleme gözetleyicisini de uzatır. Akıl yürütme tamamlamalarını, yorum `agentMessage` tamamlamalarını ve araç öncesi ham akıl yürütme veya asistan ilerlemesini otomatik bir nihai yanıt izleyebileceğinden bunlar, oturum hattını hemen serbest bırakmak yerine ilerleme sonrası yanıt korumasını kullanır. Yalnızca nihai/yorum dışı tamamlanmış `agentMessage` öğeleri ve araç öncesi ham asistan tamamlamaları asistan çıktısı serbest bırakmasını etkinleştirir: Codex daha sonra `turn/completed` olmadan sessiz kalırsa OpenClaw, mümkün olan en iyi şekilde yerel turu keser ve oturum hattını serbest bırakır. Asistan, araç, etkin öğe veya yan etki kanıtı olmadan tur tamamlama boşta kalma zaman aşımları dâhil olmak üzere yeniden oynatılması güvenli stdio uygulama sunucusu hataları, yeni bir uygulama sunucusu denemesinde bir kez yeniden denenir. Güvenli olmayan zaman aşımları yine de takılı kalan uygulama sunucusu istemcisini devreden çıkarır ve OpenClaw oturum hattını serbest bırakır. Ayrıca otomatik olarak yeniden oynatılmak yerine eski yerel iş parçacığı bağlamasını temizler. Tamamlama izleme zaman aşımları Codex'e özgü zaman aşımı metni gösterir: yeniden oynatılması güvenli durumlar yanıtın eksik olabileceğini belirtirken güvenli olmayan durumlar kullanıcıya yeniden denemeden önce mevcut durumu doğrulamasını söyler. Genel zaman aşımı tanılamaları; son uygulama sunucusu bildirim yöntemi, ham asistan yanıt öğesinin kimliği/türü/rolü, etkin istek/öğe sayıları ve etkinleştirilmiş izleme durumu gibi yapısal alanları içerir. Son bildirim ham bir asistan yanıt öğesiyse ayrıca sınırlı bir asistan metni önizlemesi içerir. Ham istem veya araç içeriğini içermez.

## Model keşfi

Codex Plugin, varsayılan olarak uygulama sunucusundan kullanılabilir modelleri ister. Model kullanılabilirliği Codex uygulama sunucusunun sorumluluğundadır; bu nedenle OpenClaw, paketlenmiş `@openai/codex` sürümünü yükselttiğinde veya bir dağıtım `appServer.command` değerini farklı bir Codex ikili dosyasına yönlendirdiğinde liste değişebilir. Kullanılabilirlik hesap kapsamında da olabilir. Bu koşum takımı ve hesaba ait canlı kataloğu görmek için çalışan bir Gateway üzerinde `/codex models` kullanın.

Keşif başarısız olursa veya zaman aşımına uğrarsa OpenClaw, paketlenmiş bir yedek katalog kullanır:

| Model kimliği       | Görünen ad | Akıl yürütme düzeyleri        |
| -------------- | ------------ | ------------------------ |
| `gpt-5.5`      | gpt-5.5      | low, medium, high, xhigh |
| `gpt-5.4-mini` | GPT-5.4-Mini | low, medium, high, xhigh |

<Note>
Geçerli paketlenmiş koşum takımı `@openai/codex` `0.145.0` sürümündedir. Bu paketlenmiş uygulama sunucusuna yönelik bir `model/list` yoklaması şu genel seçici satırlarını döndürdü:

| Model kimliği        | Girdi kipleri | Akıl yürütme düzeyleri                    |
| --------------- | ---------------- | ------------------------------------ |
| `gpt-5.6-sol`   | metin, görüntü      | low, medium, high, xhigh, max, ultra |
| `gpt-5.6-terra` | metin, görüntü      | low, medium, high, xhigh, max, ultra |
| `gpt-5.6-luna`  | metin, görüntü      | low, medium, high, xhigh, max        |
| `gpt-5.5`       | metin, görüntü      | low, medium, high, xhigh             |
| `gpt-5.2`       | metin, görüntü      | low, medium, high, xhigh             |

Uygulama sunucusu kataloğu `ultra` bildirebilir; OpenClaw akıl yürütme denetimleri şu anda `max` düzeyine kadar seçenek sunar.

Canlı seçici satırları hesap kapsamındadır ve hesaba, Codex kataloğuna veya paketlenmiş sürüme göre değişebilir; belirli bir zamana ait tabloya güvenmek yerine güncel liste için `/codex models` çalıştırın. Gizli modeller, normal model seçici seçenekleri olmadan dâhilî veya özel akışlar için uygulama sunucusu kataloğunda da görünebilir.
</Note>

Keşfi `plugins.entries.codex.config.discovery` altında ayarlayın:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          discovery: {
            enabled: true,
            timeoutMs: 2500,
          },
        },
      },
    },
  },
}
```

Başlangıcın Codex'i yoklamamasını ve yalnızca yedek kataloğu kullanmasını istediğinizde keşfi devre dışı bırakın:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          discovery: {
            enabled: false,
          },
        },
      },
    },
  },
}
```

## Çalışma alanı önyükleme dosyaları

Codex, yerel proje belgesi keşfi aracılığıyla `AGENTS.md` işlemini kendisi gerçekleştirir. Codex yedekleri yalnızca `AGENTS.md` eksik olduğunda geçerli olduğundan OpenClaw, sentetik Codex proje belgesi dosyaları yazmaz veya persona dosyaları için Codex yedek dosya adlarına bağımlı olmaz.

OpenClaw çalışma alanıyla eşdeğerlik için Codex koşum takımı, diğer önyükleme dosyalarını geliştirici talimatları olarak iletir; ancak aynı şekilde değil:

- `TOOLS.md`, **devralınan** Codex geliştirici talimatları olarak iletilir; böylece tur sırasında oluşturulan yerel Codex alt ajanları da bunları görür.
- `SOUL.md`, `IDENTITY.md` ve `USER.md`, **tur kapsamlı** iş birliği talimatları olarak iletilir. Yerel Codex alt ajanları bunları devralmaz; bu da alt ajan turlarının üst ajanın personasını ve kullanıcı profilini almasını önler.
- Yüklenen kompakt OpenClaw Skills listesi de tur kapsamlı iş birliği geliştirici talimatları olarak iletilir; dolayısıyla yerel Codex alt ajanları bunu da devralmaz.
- `HEARTBEAT.md` içeriği eklenmez; Heartbeat turları, dosya mevcut ve boş değilse dosyayı okumaya yönelik iş birliği modu işaretçisi alır.
- Yapılandırılmış ajan çalışma alanındaki `MEMORY.md` içeriği, bu çalışma alanı için bellek araçları kullanılabilir olduğunda yerel Codex tur girdisine yapıştırılmaz; içerik mevcutsa koşum takımı, tur kapsamlı iş birliği geliştirici talimatlarına küçük bir çalışma alanı belleği işaretçisi ekler ve kalıcı bellek gerekli olduğunda Codex, `memory_search` veya `memory_get` kullanmalıdır. Araçlar devre dışıysa, bellek araması kullanılamıyorsa veya etkin çalışma alanı ajan belleği çalışma alanından farklıysa `MEMORY.md`, bunun yerine normal sınırlı tur bağlamı yolunu kullanır.
- `BOOTSTRAP.md`, mevcut olduğunda OpenClaw tur girdisi referans bağlamı olarak iletilir.

## Ortam geçersiz kılmaları

Ortam geçersiz kılmaları yerel testler için kullanılabilir durumda kalır:

- `OPENCLAW_CODEX_APP_SERVER_BIN`
- `OPENCLAW_CODEX_APP_SERVER_ARGS`
- `OPENCLAW_CODEX_APP_SERVER_MODE=yolo|guardian`
- `OPENCLAW_CODEX_APP_SERVER_APPROVAL_POLICY`
- `OPENCLAW_CODEX_APP_SERVER_SANDBOX`

`OPENCLAW_CODEX_APP_SERVER_BIN`, `appServer.command` ayarlanmamışsa yönetilen ikili dosyayı atlar.

`OPENCLAW_CODEX_APP_SERVER_GUARDIAN=1` kaldırıldı. Bunun yerine `plugins.entries.codex.config.appServer.mode: "guardian"` veya tek seferlik yerel testler için `OPENCLAW_CODEX_APP_SERVER_MODE=guardian` kullanın. Yapılandırma, Plugin davranışını Codex koşum takımı kurulumunun geri kalanıyla aynı incelenmiş dosyada tuttuğu için tekrarlanabilir dağıtımlarda tercih edilir.

## İlgili

- [Codex koşum takımı](/tr/plugins/codex-harness)
- [Codex koşum takımı çalışma zamanı](/tr/plugins/codex-harness-runtime)
- [Codex gözetimi](/plugins/codex-supervision)
- [Yerel Codex Plugin'leri](/tr/plugins/codex-native-plugins)
- [Codex Bilgisayar Kullanımı](/tr/plugins/codex-computer-use)
- [OpenAI sağlayıcısı](/tr/providers/openai)
- [Yapılandırma referansı](/tr/gateway/configuration-reference)
