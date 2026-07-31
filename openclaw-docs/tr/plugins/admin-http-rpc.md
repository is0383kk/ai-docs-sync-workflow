---
read_when:
    - Gateway WebSocket RPC istemcisini kullanamayan ana makine araçları oluşturma
    - Gateway yönetim otomasyonunu özel ve güvenilir bir giriş noktasının arkasından erişime açma
    - Gateway yöntemlerine HTTP erişimi için güvenlik modelini denetleme
summary: Paketle birlikte gelen, isteğe bağlı admin-http-rpc Plugin'i üzerinden seçili Gateway kontrol düzlemi yöntemlerini kullanıma sunun
title: Yönetici HTTP RPC Plugin'i
x-i18n:
    generated_at: "2026-07-26T23:49:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0709081efd0ce65cef7edac54df9a71978cbad17e2b25df83ac9075de938376c
    source_path: plugins/admin-http-rpc.md
    workflow: 16
---

Paketle birlikte gelen `admin-http-rpc` plugin'i, Gateway WebSocket bağlantısını açık tutamayan güvenilir ana makine otomasyonları için izin verilenler listesindeki bir dizi Gateway kontrol düzlemi yöntemini HTTP üzerinden kullanıma sunar.

OpenClaw ile birlikte gelir ancak varsayılan olarak devre dışıdır; devre dışıyken rota kaydedilmez. Etkinleştirildiğinde Gateway ile aynı dinleyiciye (`http://<gateway-host>:<port>/api/v1/admin/rpc`) `POST /api/v1/admin/rpc` ekler.

Yalnızca özel ana makine araçları, tailnet otomasyonu veya güvenilir bir dahili giriş için etkinleştirin. Bu rotayı hiçbir zaman doğrudan genel internete açmayın.

## Etkinleştirmeden önce

Yönetici HTTP RPC, eksiksiz bir operatör kontrol düzlemi yüzeyidir: Gateway HTTP kimlik doğrulamasını geçen herhangi bir çağıran, aşağıdaki izin verilenler listesindeki yöntemleri çağırabilir. Yalnızca aşağıdakilerin tümü geçerliyse etkinleştirin:

- Çağıranın Gateway'i işletmesine güveniliyor.
- Çağıran WebSocket RPC istemcisini kullanamıyor.
- Rotaya yalnızca geri döngü, bir tailnet veya kimliği doğrulanmış özel bir giriş üzerinden erişilebiliyor.
- İzin verilen yöntemleri incelediniz ve bunlar çalıştırmayı planladığınız otomasyonla eşleşiyor.

Gateway WebSocket bağlantısını açık tutabilen OpenClaw istemcileri ve etkileşimli araçlar için bunun yerine WebSocket RPC kullanın.

## Etkinleştirme

Paketle birlikte gelen plugin'i etkinleştirin:

<Tabs>
  <Tab title="CLI">
    ```bash
    openclaw plugins enable admin-http-rpc
    openclaw gateway restart
    ```
  </Tab>
  <Tab title="Yapılandırma">
    ```json5
    {
      plugins: {
        entries: {
          "admin-http-rpc": { enabled: true },
        },
      },
    }
    ```
  </Tab>
</Tabs>

Rota, plugin başlatılırken kaydedilir; bu nedenle plugin yapılandırmasını değiştirdikten sonra Gateway'i yeniden başlatın.

HTTP yüzeyine artık ihtiyaç duymadığınızda devre dışı bırakın:

```bash
openclaw plugins disable admin-http-rpc
openclaw gateway restart
```

## Rotayı doğrulama

En küçük güvenli istek olarak `health` kullanın:

```bash
curl -sS http://<gateway-host>:<port>/api/v1/admin/rpc \
  -H 'Authorization: Bearer <gateway-token>' \
  -H 'Content-Type: application/json' \
  -d '{"method":"health","params":{}}'
```

Başarılı bir yanıt `ok: true` içerir:

```json
{
  "id": "generated-request-id",
  "ok": true,
  "payload": {
    "status": "ok"
  }
}
```

Plugin devre dışıyken rota, kaydedilmediği için `404` döndürür.

## Kimlik doğrulama

Plugin rotası Gateway HTTP kimlik doğrulamasını kullanır.

Yaygın kimlik doğrulama yolları:

- paylaşılan gizli anahtar kimlik doğrulaması (`gateway.auth.mode="token"` veya `"password"`): `Authorization: Bearer <token-or-password>`
- kimlik bilgisi taşıyan güvenilir HTTP kimlik doğrulaması (`gateway.auth.mode="trusted-proxy"`): yapılandırılmış kimlik duyarlı proxy üzerinden yönlendirin ve gerekli kimlik başlıklarını eklemesine izin verin
- özel girişte açık kimlik doğrulama (`gateway.auth.mode="none"`): kimlik doğrulama başlığı gerekmez

## Güvenlik modeli

Bu plugin'i eksiksiz bir Gateway operatör yüzeyi olarak değerlendirin.

- Plugin'i etkinleştirmek, izin verilenler listesindeki yönetici RPC yöntemlerine `/api/v1/admin/rpc` üzerinden kasıtlı olarak erişim sağlar.
- Plugin, ayrılmış `contracts.gatewayMethodDispatch: ["authenticated-request"]` manifest sözleşmesini bildirir; Gateway kimlik doğrulamalı HTTP rotasının kontrol düzlemi yöntemlerini işlem içinde yönlendirebilmesini sağlayan budur. Bu bir korumalı alan değildir: sözleşme, ayrılmış SDK yardımcılarının yanlışlıkla kullanılmasını önler ancak güvenilir plugin'ler yine de Gateway işlemi içinde çalışır.
- Paylaşılan gizli anahtar taşıyıcı kimlik doğrulaması (`token`/`password` modları), Gateway operatör gizli anahtarına sahip olunduğunu kanıtlar; daha dar kapsamlı `x-openclaw-scopes` başlıkları bu yolda yok sayılır ve normal tam operatör varsayılanları geri yüklenir.
- Kimlik bilgisi taşıyan güvenilir HTTP kimlik doğrulaması (`trusted-proxy` modu), mevcut olduğunda `x-openclaw-scopes` değerini dikkate alır.
- `gateway.auth.mode="none"`, plugin etkinse bu rotanın kimlik doğrulamasız olduğu anlamına gelir. Bunu yalnızca tamamen güvendiğiniz özel bir girişin arkasında kullanın.
- İstekler, plugin rotasının kimlik doğrulamasını geçtikten sonra WebSocket RPC ile aynı Gateway yöntem işleyicileri ve kapsam denetimleri üzerinden yönlendirilir.
- Rota, hazırlanmış bir askıya alma kiralaması sırasında erişilebilir kalır. Sınırlı istek doğrulaması ve yerel `commands.list` keşif yanıtı kullanılabilir durumda kalır. Gateway'e yönlendirilen yöntemlerden yalnızca `gateway.suspend.prepare`, `gateway.suspend.status` ve `gateway.suspend.resume`, kabul kapalıyken çalışabilir; izin verilenler listesindeki diğer yöntemler normal, yeniden denenebilir Gateway `UNAVAILABLE` yanıtını döndürür.
- Bu rotayı geri döngü, tailnet veya güvenilir özel bir giriş üzerinde tutun. Doğrudan genel internete açmayın. Çağıranlar güven sınırlarını aştığında ayrı Gateway'ler kullanın.

## İstek

```http
POST /api/v1/admin/rpc
Authorization: Bearer <gateway-token>
Content-Type: application/json
```

```json
{
  "id": "optional-request-id",
  "method": "health",
  "params": {}
}
```

Alanlar:

- `id` (dize, isteğe bağlı): yanıta kopyalanır. Belirtilmediğinde bir UUID oluşturulur.
- `method` (dize, zorunlu): izin verilen Gateway yöntemi adı.
- `params` (herhangi bir tür, isteğe bağlı): yönteme özgü parametreler.

Varsayılan azami istek gövdesi boyutu 1 MB'dir.

## Yanıt

Başarılı yanıtlar Gateway RPC biçimini kullanır:

```json
{
  "id": "optional-request-id",
  "ok": true,
  "payload": {}
}
```

Gateway yöntemi hataları şu biçimi kullanır:

```json
{
  "id": "optional-request-id",
  "ok": false,
  "error": {
    "code": "INVALID_REQUEST",
    "message": "bad params"
  }
}
```

HTTP durumu hata koduna göre belirlenir:

| Hata kodu                  | HTTP durumu |
| -------------------------- | ----------- |
| `INVALID_REQUEST`          | 400         |
| `APPROVAL_NOT_FOUND`       | 404         |
| `NOT_LINKED`, `NOT_PAIRED` | 409         |
| `UNAVAILABLE`              | 503         |
| `AGENT_TIMEOUT`            | 504         |
| diğer tüm kodlar           | 500         |

## İzin verilen yöntemler

- keşif: `commands.list`
  Bu plugin tarafından izin verilen HTTP RPC yöntemi adlarını döndürür.
- gateway: `health`, `status`, `logs.tail`, `usage.status`, `usage.cost`, `gateway.restart.request`, `gateway.suspend.prepare`, `gateway.suspend.status`, `gateway.suspend.resume`
- yapılandırma: `config.get`, `config.schema`, `config.schema.lookup`, `config.set`, `config.patch`, `config.apply`
- kanallar: `channels.status`, `channels.start`, `channels.stop`, `channels.logout`
- web: `web.login.start`, `web.login.wait`
- modeller: `models.list`, `models.authStatus`
- ajanlar: `agents.list`, `agents.create`, `agents.update`, `agents.delete`
- onaylar: `exec.approvals.get`, `exec.approvals.set`, `exec.approvals.node.get`, `exec.approvals.node.set`
- cron: `cron.status`, `cron.list`, `cron.get`, `cron.runs`, `cron.add`, `cron.update`, `cron.remove`, `cron.run`
- cihazlar: `device.pair.list`, `device.pair.approve`, `device.pair.reject`, `device.pair.remove`
- node'lar: `node.list`, `node.describe`, `node.pair.list`, `node.pair.approve`, `node.pair.reject`, `node.pair.remove`, `node.rename`
- görevler: `tasks.list`, `tasks.get`, `tasks.cancel`
- tanılama: `doctor.memory.status`, `update.status`

Diğer Gateway yöntemleri, kasıtlı olarak eklenene kadar engellenir.

## WebSocket karşılaştırması

Normal Gateway WebSocket RPC yolu, OpenClaw istemcileri için tercih edilen kontrol düzlemi API'si olmaya devam eder. Yönetici HTTP RPC'yi yalnızca istek/yanıt türünde bir HTTP yüzeyine ihtiyaç duyan ana makine araçları için kullanın.

Güvenilir bir cihaz kimliği olmayan paylaşılan token'lı WebSocket istemcileri, bağlantı sırasında yönetici kapsamlarını kendi kendilerine bildiremez. Yönetici HTTP RPC, mevcut güvenilir HTTP operatör modelini bilinçli olarak izler: plugin etkinleştirildiğinde, paylaşılan gizli anahtar taşıyıcı kimlik doğrulaması bu yönetici yüzeyi için tam operatör erişimi olarak değerlendirilir.

## Sorun giderme

`404 Not Found`

: Plugin devre dışıdır, etkinleştirildikten sonra Gateway yeniden başlatılmamıştır veya istek farklı bir Gateway işlemine gidiyordur.

`401 Unauthorized`

: İstek Gateway HTTP kimlik doğrulamasını karşılamadı. Taşıyıcı token'ı veya güvenilir proxy kimlik başlıklarını denetleyin.

`405 Method Not Allowed`

: İstek `POST` dışında bir şey kullandı.

`413 Payload Too Large`

: İstek gövdesi 1 MB sınırını aştı.

`400 INVALID_REQUEST`

: İstek gövdesi geçerli JSON değildir, `method` alanı eksiktir, yöntem plugin'in izin verilenler listesinde değildir veya askıya alma sürdürme kimliği etkin kiralamayla eşleşmiyordur.

`503 UNAVAILABLE`

: Gateway yöntemi başlatılıyor, hız sınırına tabi, askıya alınmış veya çakışan bir askıya alma/sürdürme işlemini bekliyor. Mevcut olduğunda `error.details` değerini inceleyin ve yeniden denemeden önce `error.retryAfterMs` değerine uyun.

## İlgili

- [Operatör kapsamları](/tr/gateway/operator-scopes)
- [Gateway güvenliği](/tr/gateway/security)
- [Uzaktan erişim](/tr/gateway/remote)
- [Plugin manifest'i](/tr/plugins/manifest#contracts-reference)
- [SDK alt yolları](/tr/plugins/sdk-subpaths)
