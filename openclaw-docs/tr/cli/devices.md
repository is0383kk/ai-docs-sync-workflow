---
read_when:
    - Cihaz eşleştirme isteklerini onaylıyorsunuz
    - Cihaz tokenlarını yenilemeniz veya iptal etmeniz gerekir
summary: '`openclaw devices` için CLI referansı (cihaz eşleştirme + token yenileme/iptal etme)'
title: Cihazlar
x-i18n:
    generated_at: "2026-07-26T23:14:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 83fb10f7a484fec06bfa5e53ae50181b12a9724746176bbace330ec468235494
    source_path: cli/devices.md
    workflow: 16
---

# `openclaw devices`

Cihaz eşleştirme isteklerini ve cihaz kapsamlı token'ları yönetin.

## Ortak seçenekler

- `--url <url>`: Gateway WebSocket URL'si (yapılandırıldığında varsayılan olarak `gateway.remote.url` kullanılır)
- `--token <token>`: Gateway token'ı (gerekiyorsa)
- `--password <password>`: Gateway parolası (parolayla kimlik doğrulama)
- `--timeout <ms>`: RPC zaman aşımı
- `--json`: JSON çıktısı (betik yazımı için önerilir)

<Warning>
`--url` ayarlandığında CLI, yapılandırma veya ortam kimlik bilgilerine geri dönmez. `--token` ya da `--password` seçeneğini açıkça geçirin; aksi takdirde komut hata verir.
</Warning>

## Komutlar

### `openclaw devices list`

Bekleyen eşleştirme isteklerini ve eşleştirilmiş cihazları listeleyin.

```bash
openclaw devices list
openclaw devices list --json
```

Önceden eşleştirilmiş bir cihaz için bekleyen bir istekte çıktı, istenen erişimi cihazın mevcut onaylanmış erişiminin yanında gösterir; böylece kapsam/rol yükseltmeleri, kaybolmuş bir eşleştirme gibi görünmek yerine görünür olur.

Eşleştirilmiş cihazların görünen adlarında şu öncelik sırası kullanılır: operatör etiketi (`devices rename` kaynağındaki `operatorLabel`), ardından istemci `displayName`, ardından `clientId`, ardından `deviceId`.

### `openclaw devices approve [requestId] [--latest]`

Bekleyen bir eşleştirme isteğini tam `requestId` değeriyle onaylayın. `requestId` belirtilmemesi veya `--latest` geçirilmesi yalnızca en yeni bekleyen isteğin önizlemesini gösterir ve çıkar (kod 1); onaylamak için komutu tam istek kimliğiyle yeniden çalıştırın.

```bash
openclaw devices approve
openclaw devices approve <requestId>
openclaw devices approve --latest
```

<Note>
Bir cihaz değiştirilmiş kimlik doğrulama ayrıntılarıyla (rol, kapsamlar veya genel anahtar) eşleştirmeyi yeniden denerse OpenClaw, önceki bekleyen girdinin yerine yeni bir `requestId` koyar. Geçerli kimliği almak için onaylamadan hemen önce `openclaw devices list` komutunu çalıştırın.
</Note>

Onay davranışı:

- Cihaz zaten eşleştirilmişse ve daha geniş kapsamlar veya rol istiyorsa OpenClaw mevcut onayı korur ve yeni bir bekleyen yükseltme isteği oluşturur. Onaylamadan önce `openclaw devices list` içindeki `Requested` ile `Approved` değerlerini karşılaştırın veya `--latest` ile önizleyin.
- Bir `node` rolünü veya operatör dışındaki başka bir rolü onaylamak için `operator.admin` gerekir. Operatör cihazı onayları için `operator.pairing` yeterlidir, ancak yalnızca istenen operatör kapsamları çağıranın kendi kapsamları içinde kaldığında. Bkz. [Operatör kapsamları](/tr/gateway/operator-scopes).
- `gateway.nodes.pairing.autoApproveCidrs` yapılandırılmışsa eşleşen istemci IP'lerinden gelen ilk `role: node` istekleri, bu listede görünmeden önce otomatik olarak onaylanabilir. Varsayılan olarak devre dışıdır; operatör/tarayıcı istemcilerine veya yükseltme isteklerine hiçbir zaman uygulanmaz.
- `gateway.nodes.pairing.sshVerify` (varsayılan olarak açık), gateway cihaz anahtarını SSH üzerinden Node ana makinesinde doğruladığında ilk `role: node` isteklerini otomatik olarak onaylar. Bu nedenle istekler, göründükten kısa süre sonra onaylanmış duruma geçebilir. SSH doğrulamasını devre dışı bırakmak için `sshVerify: false` ayarını yapın; bu, `autoApproveCidrs` ayarından bağımsızdır, dolayısıyla yalnızca elle eşleştirme için onu da kaldırın.

### `openclaw devices reject <requestId>`

Bekleyen bir cihaz eşleştirme isteğini reddedin.

```bash
openclaw devices reject <requestId>
```

### `openclaw devices remove <deviceId>`

Eşleştirilmiş bir cihaz girdisini kaldırın.

```bash
openclaw devices remove <deviceId>
openclaw devices remove <deviceId> --json
```

Eşleştirilmiş cihaz token'ıyla kimliği doğrulanan bir çağıran yalnızca **kendi** cihaz girdisini kaldırabilir. Başka bir cihazı kaldırmak için `operator.admin` gerekir.

### `openclaw devices rename --device <id> --name <label>`

Eşleştirilmiş bir cihaza operatör etiketi atayın. Etiketler sahip tarafındaki durumdur: eşleştirme onarımlarından ve rol yeniden onaylarından sonra korunurlar ve kararlı `deviceId` değerini değiştirmezler.

```bash
openclaw devices rename --device <deviceId> --name "Kitchen Mac"
openclaw devices rename --device <deviceId> --name "Kitchen Mac" --json
```

- `--name` zorunludur, başındaki ve sonundaki boşluklar kırpılır, boş olamaz ve en fazla 64 karakter olabilir.
- Görüntüleme yüzeyleri (CLI listesi, Control UI envanteri), istemcinin bildirdiği görünen ad yerine operatör etiketini tercih eder.
- Yönetici olmayan eşleştirilmiş cihaz çağıranı yalnızca **kendi** cihazını yeniden adlandırabilir. Başka bir cihazı yeniden adlandırmak için `operator.admin` gerekir.

### `openclaw devices clear --yes [--pending]`

Eşleştirilmiş cihazları toplu olarak temizleyin. `--yes` tarafından kısıtlanır.

```bash
openclaw devices clear --yes
openclaw devices clear --yes --pending
openclaw devices clear --yes --pending --json
```

`--pending`, bekleyen tüm eşleştirme isteklerini de reddeder.

### `openclaw devices rotate --device <id> --role <role> [--scope <scope...>]`

Bir role ait cihaz token'ını döndürün ve isteğe bağlı olarak kapsamlarını güncelleyin.

```bash
openclaw devices rotate --device <deviceId> --role operator --scope operator.read --scope operator.write
```

- Hedef rol, söz konusu cihazın onaylanmış eşleştirme sözleşmesinde zaten bulunmalıdır; döndürme, onaylanmamış yeni bir rol oluşturamaz.
- `--scope` belirtilmezse sonraki yeniden bağlantılarda saklanan token'ın önbelleğe alınmış onaylı kapsamları yeniden kullanılır. Açık `--scope` değerlerinin geçirilmesi, gelecekteki önbelleğe alınmış token yeniden bağlantıları için saklanan kapsam kümesinin yerini alır.
- Yönetici olmayan eşleştirilmiş cihaz çağıranı yalnızca **kendi** cihaz token'ını döndürebilir ve hedef kapsam kümesi çağıranın kendi operatör kapsamları içinde kalmalıdır; döndürme, çağıranın hâlihazırda sahip olduğundan daha geniş bir token oluşturamaz veya koruyamaz.

Döndürme meta verilerini JSON olarak döndürür. Çağıran, ilgili cihaz token'ıyla kimliği doğrulanmışken kendi token'ını döndürürse yanıt, istemcinin yeniden bağlanmadan önce kalıcı olarak saklayabilmesi için yedek token'ı içerir. Paylaşımlı/yönetici döndürmeleri bearer token'ı hiçbir zaman yanıtta göstermez.

### `openclaw devices revoke --device <id> --role <role>`

Bir role ait cihaz token'ını iptal edin.

```bash
openclaw devices revoke --device <deviceId> --role node
```

Yönetici olmayan eşleştirilmiş cihaz çağıranı yalnızca **kendi** cihaz token'ını iptal edebilir. Başka bir cihazın token'ını iptal etmek için `operator.admin` gerekir. Hedef kapsam kümesi de çağıranın kendi operatör kapsamları içinde olmalıdır; yalnızca eşleştirme yetkisine sahip çağıranlar yönetici/yazma operatörü token'larını iptal edemez.

## Notlar

- Bu komutlar `operator.pairing` (veya `operator.admin`) kapsamını gerektirir. Operatör dışındaki cihaz rolleri her zaman `operator.admin` gerektirir; bkz. [Operatör kapsamları](/tr/gateway/operator-scopes).
- Token döndürme ve iptal işlemleri, cihazın onaylanmış eşleştirme rol kümesi ve kapsam temel çizgisi içinde kalır. Bağımsız bir önbelleğe alınmış token girdisi, token yönetimi hedefi için yetki vermez.
- Eşleştirilmiş cihaz token'ı oturumlarında cihazlar arası yönetim (`remove`, `rename`, `rotate`, `revoke`), çağıranda `operator.admin` bulunmadıkça yalnızca kendi cihazıyla sınırlıdır.
- Token döndürme yeni bir token (hassas) döndürür — bunu bir gizli bilgi gibi ele alın.
- Yerel geri döngüde eşleştirme kapsamı kullanılamıyorsa ve açık bir `--url` geçirilmemişse `list`/`approve`, yerel eşleştirme durumuna geri dönebilir.

## Token sapmasını giderme kontrol listesi

Control UI veya diğer istemciler `AUTH_TOKEN_MISMATCH`, `AUTH_DEVICE_TOKEN_MISMATCH` ya da `AUTH_SCOPE_MISMATCH` nedeniyle sürekli başarısız olduğunda bunu kullanın.

1. Geçerli gateway token kaynağını doğrulayın:

   ```bash
   openclaw config get gateway.auth.token
   ```

2. Eşleştirilmiş cihazları listeleyin ve etkilenen cihaz kimliğini belirleyin:

   ```bash
   openclaw devices list
   ```

3. Etkilenen cihazın operatör token'ını döndürün:

   ```bash
   openclaw devices rotate --device <deviceId> --role operator
   ```

4. Döndürme yeterli olmazsa eski eşleştirmeyi kaldırın ve yeniden onaylayın:

   ```bash
   openclaw devices remove <deviceId>
   openclaw devices list
   openclaw devices approve <requestId>
   ```

5. İstemci bağlantısını geçerli paylaşılan token/parolayla yeniden deneyin.

Notlar:

- Normal yeniden bağlantı kimlik doğrulama önceliği: önce açıkça belirtilen paylaşılan token/parola, ardından açık `deviceToken`, ardından saklanan cihaz token'ı, ardından önyükleme token'ı.
- Güvenilir `AUTH_TOKEN_MISMATCH` kurtarması, sınırlı tek bir yeniden deneme için paylaşılan token ile saklanan cihaz token'ını geçici olarak birlikte gönderebilir.
- `AUTH_SCOPE_MISMATCH`, cihaz token'ının tanındığı ancak istenen kapsam kümesini taşımadığı anlamına gelir; paylaşılan gateway kimlik doğrulamasını değiştirmeden önce eşleştirme/kapsam onayı sözleşmesini düzeltin.

İlgili:

- [Dashboard kimlik doğrulama sorunlarını giderme](/tr/web/dashboard#if-you-see-unauthorized-1008)
- [Gateway sorunlarını giderme](/tr/gateway/troubleshooting#dashboard-control-ui-connectivity)

## Paperclip / `openclaw_gateway` ilk çalıştırma onayı

`openclaw_gateway` bağdaştırıcısı üzerinden bağlanan Paperclip ajanları, diğer tüm yeni istemcilerle aynı ilk çalıştırma cihaz eşleştirme onayından geçer. Paperclip `openclaw_gateway_pairing_required` bildirirse bekleyen cihazı onaylayın ve yeniden deneyin.

```bash
openclaw devices approve --latest
```

Önizleme, tam `openclaw devices approve <requestId>` komutunu yazdırır; ayrıntıları doğrulayın, ardından onaylamak için bu komutu istek kimliğiyle yeniden çalıştırın. Uzak bir gateway veya açık kimlik bilgileri için önizleme ve onaylama sırasında aynı seçenekleri geçirin:

```bash
openclaw devices approve --latest --url <gateway-ws-url> --token <gateway-token>
```

Her yeniden başlatmadan sonra yeniden onaylamayı önlemek için Paperclip'in her çalıştırmada yeni bir geçici cihaz kimliği oluşturmasına izin vermek yerine kalıcı bir `adapterConfig.devicePrivateKeyPem` yapılandırın:

```json
{
  "adapterConfig": {
    "devicePrivateKeyPem": "<ed25519-private-key-pkcs8-pem>"
  }
}
```

Onay sürekli başarısız oluyorsa bekleyen bir isteğin bulunduğunu doğrulamak için önce `openclaw devices list` komutunu çalıştırın.

## İlgili

- [CLI başvurusu](/tr/cli)
- [Node'lar](/tr/nodes)
