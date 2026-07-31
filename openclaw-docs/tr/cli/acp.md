---
read_when:
    - ACP tabanlı IDE entegrasyonlarını ayarlama
    - ACP oturumlarının Gateway'e yönlendirilmesinde hata ayıklama
summary: IDE entegrasyonları için ACP köprüsünü çalıştırın
title: ACP
x-i18n:
    generated_at: "2026-07-26T23:51:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: becdcfdd1cc62b206cc92e9b8248c79a2ff63cfc3779d8a124b9713e779ad33c
    source_path: cli/acp.md
    workflow: 16
---

Bir OpenClaw Gateway ile iletişim kuran [Agent Client Protocol (ACP)](https://agentclientprotocol.com/) köprüsünü çalıştırın.

`openclaw acp`, IDE'ler için stdio üzerinden ACP iletişimi kurar ve istemleri WebSocket üzerinden Gateway'e iletirken ACP oturumlarını Gateway oturum anahtarlarıyla eşlenmiş hâlde tutar. Tam bir ACP yerel düzenleyici çalışma zamanı değil, Gateway destekli bir ACP köprüsüdür: oturum yönlendirmeye, istem teslimine ve akış güncellemelerine odaklanır.

Harici bir MCP istemcisinin ACP yürütme ortamı oturumu barındırmak yerine doğrudan OpenClaw kanal konuşmalarıyla iletişim kurmasını istiyorsanız [`openclaw mcp serve`](/tr/cli/mcp) kullanın.

## Bu ne değildir?

`openclaw acp`, OpenClaw'ın bir ACP sunucusu olarak çalıştığı anlamına gelir: Bir IDE veya ACP istemcisi OpenClaw'a bağlanır ve OpenClaw bu işi bir Gateway oturumuna iletir.

Bu, OpenClaw'ın Codex veya Claude Code gibi harici bir yürütme ortamını `acpx` üzerinden çalıştırdığı [ACP Aracılarından](/tr/tools/acp-agents) farklıdır.

Kısa kural:

- düzenleyici/istemci, OpenClaw ile ACP üzerinden iletişim kurmak istiyorsa: `openclaw acp` kullanın
- OpenClaw, Codex/Claude/Gemini'yi bir ACP yürütme ortamı olarak başlatacaksa: `/acp spawn` ve [ACP Aracılarını](/tr/tools/acp-agents) kullanın

## Uyumluluk matrisi

| ACP alanı                                                              | Durum      | Notlar                                                                                                                                                                                                                                 |
| --------------------------------------------------------------------- | ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `initialize`, `newSession`, `prompt`, `cancel`                        | Uygulandı | stdio'dan Gateway chat/send + abort'a uzanan temel köprü akışı.                                                                                                                                                                             |
| `listSessions`, eğik çizgi komutları                                        | Uygulandı | Oturum listesi, sınırlı imleç sayfalaması ve Gateway oturum satırlarının çalışma alanı meta verilerini taşıdığı durumlarda `cwd` filtrelemesiyle Gateway oturum durumuna göre çalışır; komutlar `available_commands_update` aracılığıyla duyurulur.                     |
| Oturum köken zinciri meta verileri                                              | Uygulandı | Oturum listeleri ve oturum bilgisi anlık görüntüleri, `_meta` içinde OpenClaw üst ve alt köken zincirini içerir; böylece ACP istemcileri özel Gateway yan kanalları olmadan alt aracı grafiklerini oluşturabilir.                                                     |
| `resumeSession`, `closeSession`                                       | Uygulandı | Sürdürme, geçmişi yeniden oynatmadan bir ACP oturumunu mevcut bir Gateway oturumuna yeniden bağlar. Kapatma, etkin köprü çalışmasını iptal eder, bekleyen istemleri iptal edilmiş olarak sonuçlandırır ve köprü oturumu durumunu serbest bırakır.                                   |
| `loadSession`                                                         | Kısmi     | ACP oturumunu bir Gateway oturum anahtarına yeniden bağlar ve köprü tarafından oluşturulan oturumlar için ACP olay defteri geçmişini yeniden oynatır. Daha eski veya deftersiz oturumlar, depolanan kullanıcı/asistan metnine geri döner.                                                  |
| İstem içeriği (`text`, gömülü `resource`, görseller)                  | Kısmi     | Metin/kaynaklar sohbet girdisine düzleştirilir; görseller Gateway eklerine dönüşür.                                                                                                                                                            |
| Oturum modları                                                         | Kısmi     | `session/set_mode` desteklenir; köprü, düşünme düzeyi, araç ayrıntı düzeyi, akıl yürütme, kullanım ayrıntısı ve yükseltilmiş eylemler için Gateway destekli oturum denetimleri sunar. Daha geniş ACP yerel mod/yapılandırma yüzeyleri hâlâ kapsam dışındadır. |
| Düşünce akışı                                                     | Uygulandı | Modelin düşünme içeriği, `agent_thought_chunk` oturum güncellemeleri olarak aktarılır. ACP yerel oturum planları yayımlanmaz.                                                                                                                    |
| Oturum bilgisi ve kullanım güncellemeleri                                        | Kısmi     | Köprü, önbelleğe alınmış Gateway oturumu anlık görüntülerinden `session_info_update` ve azami gayretle `usage_update` bildirimleri yayımlar. Kullanım yaklaşık değerdir ve yalnızca Gateway belirteç toplamları güncel olarak işaretlendiğinde gönderilir.                             |
| Araç akışı                                                        | Kısmi     | `tool_call`/`tool_call_update` olayları, ham G/Ç'yi, metin içeriğini ve Gateway araç bağımsız değişkenleri/sonuçları bunları sunduğunda azami gayretle dosya konumlarını içerir. Gömülü terminaller ve daha zengin, fark odaklı çıktılar sunulmaz.                     |
| Yürütme onayları                                                        | Kısmi     | Etkin ACP istem turları sırasında Gateway yürütme onayı istemleri, `session/request_permission` ile ACP istemcisine aktarılır.                                                                                                               |
| Oturum başına MCP sunucuları (`mcpServers`)                                | Desteklenmiyor | Köprü modu, oturum başına MCP sunucusu isteklerini reddeder. Bunun yerine MCP'yi OpenClaw Gateway veya aracı üzerinde yapılandırın.                                                                                                                          |
| İstemci dosya sistemi yöntemleri (`fs/read_text_file`, `fs/write_text_file`) | Desteklenmiyor | Köprü, ACP istemcisi dosya sistemi yöntemlerini çağırmaz.                                                                                                                                                                               |
| İstemci terminal yöntemleri (`terminal/*`)                                | Desteklenmiyor | Köprü, ACP istemci terminalleri oluşturmaz veya terminal kimliklerini araç çağrıları üzerinden aktarmayı gerçekleştirmez.                                                                                                                                            |

## Bilinen sınırlamalar

- `loadSession`, tam ACP olay defteri geçmişini yalnızca köprü tarafından oluşturulan oturumlar için yeniden oynatır. Daha eski veya deftersiz oturumlar transkript geri dönüşünü kullanır ve geçmiş araç çağrılarını ya da sistem bildirimlerini yeniden oluşturmaz.
- Birden çok ACP istemcisi aynı Gateway oturum anahtarını paylaşırsa olay ve iptal yönlendirmesi, istemci başına kesin olarak yalıtılmak yerine azami gayretle gerçekleştirilir. Temiz, düzenleyiciye yerel turlara ihtiyacınız varsa varsayılan yalıtılmış `acp-bridge:<uuid>` oturumlarını tercih edin.
- Gateway durdurma durumları ACP durdurma nedenlerine çevrilir, ancak bu eşleme tam ACP yerel çalışma zamanından daha az ifade gücüne sahiptir.
- Oturum denetimleri, Gateway ayarlarının odaklanmış bir alt kümesini sunar: düşünme düzeyi, araç ayrıntı düzeyi, akıl yürütme, kullanım ayrıntısı ve yükseltilmiş eylemler. Model seçimi ve yürütme ana makinesi denetimleri ACP yapılandırma seçenekleri olarak sunulmaz.
- `session_info_update` ve `usage_update`, canlı ACP yerel çalışma zamanı muhasebesinden değil, Gateway oturumu anlık görüntülerinden türetilir. Kullanım yaklaşık değerdir, maliyet verisi içermez ve yalnızca Gateway toplam belirteç verilerini güncel olarak işaretlediğinde yayımlanır.
- Araç izleme verileri azami gayret esasına dayanır: Köprü, bilinen araç bağımsız değişkenlerinde/sonuçlarında görünen dosya yollarını sunar, ancak ACP terminalleri veya yapılandırılmış dosya farkları yayımlamaz.
- Yürütme onayı aktarımı etkin ACP istem turuyla sınırlıdır; diğer Gateway oturumlarından gelen onaylar yok sayılır.

## Kullanım

```bash
openclaw acp

# Uzak Gateway
openclaw acp --url wss://gateway-host:18789 --token <token>

# Uzak Gateway (dosyadan belirteç)
openclaw acp --url wss://gateway-host:18789 --token-file ~/.openclaw/gateway.token

# Mevcut bir oturum anahtarına bağlan
openclaw acp --session agent:main:main

# Etikete göre bağlan (önceden mevcut olmalıdır)
openclaw acp --session-label "support inbox"

# İlk istemden önce oturum anahtarını sıfırla
openclaw acp --session agent:main:main --reset-session
```

## ACP istemcisi (hata ayıklama)

Köprünün temel doğrulamasını IDE olmadan yapmak için yerleşik ACP istemcisini kullanın. ACP köprüsünü başlatır ve etkileşimli olarak istemler yazmanıza olanak tanır.

```bash
openclaw acp client

# Başlatılan köprüyü uzak bir Gateway'e yönlendir
openclaw acp client --server-args --url wss://gateway-host:18789 --token-file ~/.openclaw/gateway.token

# Sunucu komutunu geçersiz kıl (varsayılan: openclaw)
openclaw acp client --server "node" --server-args openclaw.mjs acp --url ws://127.0.0.1:19001
```

İzin modeli (istemci hata ayıklama modu):

- Otomatik onay, izin listesine dayalıdır ve yalnızca güvenilir çekirdek araç kimliklerine uygulanır.
- `read` otomatik onayı geçerli çalışma diziniyle (`--cwd` ayarlandığında) sınırlıdır.
- ACP yalnızca dar salt okunur sınıfları otomatik onaylar: etkin cwd altındaki kapsamlı `read` çağrıları ve salt okunur arama araçları (`search`, `web_search`, `memory_search`). Bilinmeyen/çekirdek dışı araçlar, kapsam dışı okumalar, yürütme yetenekli araçlar, denetim düzlemi araçları, değişiklik yapan araçlar ve etkileşimli akışlar her zaman açık istem onayı gerektirir.
- Sunucu tarafından sağlanan `toolCall.kind`, yetkilendirme kaynağı değil, güvenilmeyen meta veri olarak değerlendirilir.
- Bu ACP köprüsü ilkesi, ACPX yürütme ortamı izinlerinden ayrıdır. OpenClaw'ı `acpx` arka ucu üzerinden çalıştırırsanız `plugins.entries.acpx.config.permissionMode=approve-all`, söz konusu yürütme ortamı oturumu için acil durum "yolo" anahtarıdır.

## Protokol duman testi

Protokol düzeyinde hata ayıklama için yalıtılmış durumla bir Gateway başlatın ve ACP JSON-RPC istemcisiyle stdio üzerinden `openclaw acp` çalıştırın. Mutlak bir `cwd` ile `initialize`, `session/new`, `session/list`, `session/resume`, `session/close`, yinelenen kapatma ve eksik sürdürme durumlarını kapsayın.

Kanıt; duyurulan yaşam döngüsü yeteneklerini, Gateway destekli bir oturum satırını, güncelleme bildirimlerini ve Gateway `sessions.list` günlüğünü içermelidir:

```json
{
  "initialize": {
    "protocolVersion": 1,
    "agentCapabilities": {
      "sessionCapabilities": {
        "list": {},
        "resume": {},
        "close": {}
      }
    }
  },
  "listSessions": {
    "sessions": [
      {
        "sessionId": "agent:main:acp-smoke",
        "cwd": "/path/to/workspace",
        "_meta": {
          "sessionKey": "agent:main:acp-smoke",
          "kind": "direct"
        }
      }
    ],
    "nextCursor": null
  },
  "notifications": ["session_info_update", "available_commands_update", "usage_update"],
  "gatewayLogTail": ["[gateway] ready", "[ws] ⇄ res ✓ sessions.list 305ms"]
}
```

Tek ACP kanıtı olarak `openclaw gateway call sessions.list` kullanmaktan kaçının. Bu CLI yolu, yeni belirteçli bir operatör kapsamı yükseltmesi isteyebilir; ACP köprüsünün doğruluğu, ACP stdio çerçeveleri ve Gateway `sessions.list` günlüğüyle kanıtlanır.

## Nasıl kullanılır?

Bir IDE (veya başka bir istemci) Agent Client Protocol kullanıyorsa ve bu istemcinin bir OpenClaw Gateway oturumunu yönetmesini istiyorsanız ACP kullanın.

1. Gateway'in çalıştığından emin olun (yerel veya uzak).
2. Gateway hedefini yapılandırın (yapılandırma veya bayraklar).
3. IDE'nizi stdio üzerinden `openclaw acp` çalıştıracak şekilde ayarlayın.

Örnek yapılandırma (kalıcı):

```bash
openclaw config set gateway.remote.url wss://gateway-host:18789
openclaw config set gateway.remote.token <token>
```

Doğrudan çalıştırma örneği (yapılandırmaya yazmaz):

```bash
openclaw acp --url wss://gateway-host:18789 --token <token>
# yerel işlem güvenliği için tercih edilir
openclaw acp --url wss://gateway-host:18789 --token-file ~/.openclaw/gateway.token
```

## Aracıları seçme

ACP, aracıları doğrudan seçmez. Gateway oturum anahtarına göre yönlendirme yapar. Belirli bir aracıyı hedeflemek için aracı kapsamlı oturum anahtarlarını kullanın:

```bash
openclaw acp --session agent:main:main
openclaw acp --session agent:design:main
openclaw acp --session agent:qa:bug-123
```

Her ACP oturumu tek bir Gateway oturum anahtarıyla eşlenir. Bir aracının birçok oturumu olabilir; anahtarı veya etiketi geçersiz kılmadığınız sürece ACP, varsayılan olarak yalıtılmış bir `acp-bridge:<uuid>` oturumu kullanır.

Oturum başına `mcpServers`, köprü modunda desteklenmez. Bir ACP istemcisi bunları `newSession` veya `loadSession` sırasında gönderirse köprü, sessizce yok saymak yerine açık bir hata döndürür.

ACPX destekli oturumların OpenClaw Plugin araçlarını veya `cron` gibi seçili yerleşik araçları görmesini istiyorsanız oturum başına `mcpServers` iletmeye çalışmak yerine Gateway tarafındaki ACPX MCP köprülerini etkinleştirin. Bkz. [ACP Aracıları](/tr/tools/acp-agents-setup#plugin-tools-mcp-bridge) ve [OpenClaw araçları MCP köprüsü](/tr/tools/acp-agents-setup#openclaw-tools-mcp-bridge).

## `acpx` üzerinden kullanım (Codex, Claude ve diğer ACP istemcileri)

Codex veya Claude Code gibi bir kodlama aracısının ACP üzerinden OpenClaw botunuzla iletişim kurmasını istiyorsanız yerleşik `openclaw` hedefiyle birlikte `acpx` kullanın.

Tipik akış:

1. Gateway'i çalıştırın ve ACP köprüsünün ona erişebildiğinden emin olun.
2. `acpx openclaw` öğesini `openclaw acp` hedefine yönlendirin.
3. Kodlama aracısının kullanmasını istediğiniz OpenClaw oturum anahtarını hedefleyin.

Örnekler:

```bash
# Varsayılan OpenClaw ACP oturumunuza tek seferlik istek
acpx openclaw exec "Etkin OpenClaw oturum durumunu özetle."

# Sonraki etkileşimler için kalıcı, adlandırılmış oturum
acpx openclaw sessions ensure --name codex-bridge
acpx openclaw -s codex-bridge --cwd /path/to/repo \
  "OpenClaw çalışma aracımdan bu depoyla ilgili yakın tarihli bağlamı iste."
```

`acpx openclaw` öğesinin her seferinde belirli bir Gateway'i ve oturum anahtarını hedeflemesini istiyorsanız `~/.acpx/config.json` içindeki `openclaw` aracı komutunu geçersiz kılın:

```json
{
  "agents": {
    "openclaw": {
      "command": "env OPENCLAW_HIDE_BANNER=1 OPENCLAW_SUPPRESS_NOTES=1 openclaw acp --url ws://127.0.0.1:18789 --token-file ~/.openclaw/gateway.token --session agent:main:main"
    }
  }
}
```

Depoya yerel bir OpenClaw çalışma kopyası için ACP akışının temiz kalması amacıyla geliştirme çalıştırıcısı yerine doğrudan CLI giriş noktasını kullanın:

```bash
env OPENCLAW_HIDE_BANNER=1 OPENCLAW_SUPPRESS_NOTES=1 node openclaw.mjs acp ...
```

Bu, Codex, Claude Code veya ACP destekli başka bir istemcinin terminali taramadan bir OpenClaw aracısından bağlamsal bilgi almasını sağlamanın en kolay yoludur.

## Zed düzenleyicisini ayarlama

`~/.config/zed/settings.json` içine özel bir ACP aracısı ekleyin (veya Zed'in Settings arayüzünü kullanın):

```json
{
  "agent_servers": {
    "OpenClaw ACP": {
      "type": "custom",
      "command": "openclaw",
      "args": ["acp"],
      "env": {}
    }
  }
}
```

Belirli bir Gateway'i veya aracıyı hedeflemek için:

```json
{
  "agent_servers": {
    "OpenClaw ACP": {
      "type": "custom",
      "command": "openclaw",
      "args": [
        "acp",
        "--url",
        "wss://gateway-host:18789",
        "--token",
        "<token>",
        "--session",
        "agent:design:main"
      ],
      "env": {}
    }
  }
}
```

Zed'de Agent panelini açın ve bir ileti dizisi başlatmak için "OpenClaw ACP" öğesini seçin.

## Oturum eşleme

Varsayılan olarak ACP köprü oturumları, `acp-bridge:` ön ekine sahip yalıtılmış bir Gateway oturum anahtarı alır. Bu normal model köprü oturumları sentetik ve geçicidir: eski girdi temizliğine tabidir ve korunan insan konuşması yüzeyleri olarak değerlendirilmez. Bilinen bir oturumu yeniden kullanmak için bir oturum anahtarı veya etiketi iletin:

- `--session <key>`: belirli bir Gateway oturum anahtarı kullanır.
- `--session-label <label>`: mevcut bir oturumu etikete göre çözümler.
- `--reset-session`: bu anahtar için yeni bir oturum kimliği oluşturur (aynı anahtar, yeni döküm).

ACP istemciniz meta verileri destekliyorsa oturum başına geçersiz kılabilirsiniz:

```json
{
  "_meta": {
    "sessionKey": "agent:main:main",
    "sessionLabel": "support inbox",
    "resetSession": true
  }
}
```

Oturum anahtarları hakkında daha fazla bilgi için bkz. [/concepts/session](/tr/concepts/session).

## Seçenekler

- `--url <url>`: Gateway WebSocket URL'si (yapılandırılmışsa varsayılan değer: `gateway.remote.url`).
- `--token <token>`: Gateway kimlik doğrulama belirteci.
- `--token-file <path>`: Gateway kimlik doğrulama belirtecini dosyadan okur.
- `--password <password>`: Gateway kimlik doğrulama parolası.
- `--password-file <path>`: Gateway kimlik doğrulama parolasını dosyadan okur.
- `--session <key>`: varsayılan oturum anahtarı.
- `--session-label <label>`: çözümlenecek varsayılan oturum etiketi.
- `--require-existing`: oturum anahtarı/etiketi yoksa başarısız olur.
- `--reset-session`: ilk kullanımdan önce oturum anahtarını sıfırlar.
- `--no-prefix-cwd`: istemlerin başına çalışma dizinini eklemez.
- `--provenance <off|meta|meta+receipt>`: ACP kaynak meta verilerini veya alındı bilgilerini içerir.
- `--verbose, -v`: stderr'e ayrıntılı günlük kaydı yapar.

Güvenlik notu:

- `--token` ve `--password`, bazı sistemlerde yerel işlem listelerinde görülebilir. `--token-file`/`--password-file` veya ortam değişkenlerini (`OPENCLAW_GATEWAY_TOKEN`, `OPENCLAW_GATEWAY_PASSWORD`) tercih edin.
- Gateway kimlik doğrulama çözümlemesi, diğer Gateway istemcilerinin kullandığı ortak sözleşmeyi izler:
  - yerel mod: ortam (`OPENCLAW_GATEWAY_*`), ardından `gateway.auth.*`; yalnızca `gateway.auth.*` ayarlanmamışsa `gateway.remote.*` değerine geri döner (yapılandırılmış ancak çözümlenemeyen yerel SecretRef, sessizce geri dönmek yerine güvenli biçimde başarısız olur)
  - uzak mod: uzak öncelik kurallarına göre ortam/yapılandırma geri dönüşüyle `gateway.remote.*`
  - `--url`, geçersiz kılmaya karşı güvenlidir ve örtük yapılandırma/ortam kimlik bilgilerini yeniden kullanmaz; açıkça `--token`/`--password` (veya dosya çeşitlerini) iletin

### `acp client` seçenekleri

- `--cwd <dir>`: ACP oturumunun çalışma dizini.
- `--server <command>`: ACP sunucu komutu (varsayılan: `openclaw`).
- `--server-args <args...>`: ACP sunucusuna iletilen ek bağımsız değişkenler.
- `--server-verbose`: ACP sunucusunda ayrıntılı günlük kaydını etkinleştirir.
- `--verbose, -v`: ayrıntılı istemci günlük kaydı.
- `openclaw acp client`, başlatılan köprü işleminde `OPENCLAW_SHELL=acp-client` değerini ayarlar; bu değer bağlama özgü kabuk/profil kuralları için kullanılabilir.

## İlgili

- [CLI başvurusu](/tr/cli)
- [ACP aracıları](/tr/tools/acp-agents)
