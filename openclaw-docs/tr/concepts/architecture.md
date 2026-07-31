---
read_when:
    - Gateway protokolü, istemciler veya aktarımlar üzerinde çalışma
summary: WebSocket Gateway mimarisi, bileşenleri ve istemci akışları
title: Gateway mimarisi
x-i18n:
    generated_at: "2026-07-26T22:42:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f8054bd87f738b957c24f8d6965d55365de2293d44902530a9ba778afa597cc7
    source_path: concepts/architecture.md
    workflow: 16
---

## Genel Bakış

- Uzun süre çalışan tek bir **Gateway**, tüm mesajlaşma yüzeylerini yönetir (Baileys üzerinden WhatsApp, grammY üzerinden Telegram, Slack, Discord, Signal, iMessage, WebChat).
- Denetim düzlemi istemcileri (macOS uygulaması, CLI, web kullanıcı arayüzü, otomasyonlar), yapılandırılmış bağlama ana bilgisayarındaki **WebSocket** üzerinden Gateway'e bağlanır (varsayılan: `127.0.0.1:18789`).
- **Node'lar** (macOS/iOS/Android/başsız) da **WebSocket** üzerinden bağlanır, ancak açık yetenekler/komutlarla `role: node` bildirir.
- Her ana bilgisayar için bir Gateway bulunur; WhatsApp oturumunu açan tek yer burasıdır.
- **Tuval ana bilgisayarı**, Gateway HTTP sunucusu tarafından şu yollar altında sunulur:
  - `/__openclaw__/canvas/` (agent tarafından düzenlenebilir HTML/CSS/JS)
  - `/__openclaw__/a2ui/` (A2UI ana bilgisayarı)

  Gateway ile aynı bağlantı noktasını kullanır (varsayılan: `18789`).

## Bileşenler ve akışlar

### Gateway (arka plan hizmeti)

- Sağlayıcı bağlantılarını sürdürür.
- Türü belirlenmiş bir WS API'si sunar (istekler, yanıtlar, sunucudan gönderilen olaylar).
- Gelen çerçeveleri JSON Schema'ya göre doğrular.
- `agent`, `chat`, `presence`, `health`, `heartbeat`, `cron` gibi olaylar yayar.

### İstemciler (Mac uygulaması / CLI / web yönetimi)

- Her istemci için bir WS bağlantısı.
- İstek gönderir (`health`, `status`, `send`, `agent`, `system-presence`).
- Olaylara abone olur (`tick`, `agent`, `presence`, `shutdown`).

### Node'lar (macOS / iOS / Android / başsız)

- `role: node` ile **aynı WS sunucusuna** bağlanır.
- `connect` içinde bir cihaz kimliği sağlar; eşleştirme **cihaz tabanlıdır** (rol: `node`) ve onay, cihaz eşleştirme deposunda tutulur.
- `canvas.*`, `camera.*`, `screen.record`, `location.get` gibi komutları kullanıma sunar.

Protokol ayrıntıları: [Gateway protokolü](/tr/gateway/protocol)

### WebChat

- Sohbet geçmişi ve gönderimler için Gateway WS API'sini kullanan statik kullanıcı arayüzü.
- Uzak kurulumlarda diğer istemcilerle aynı SSH/Tailscale tüneli üzerinden bağlanır.

## Bağlantı yaşam döngüsü (tek istemci)

```mermaid
sequenceDiagram
    participant İstemci
    participant Gateway

    İstemci->>Gateway: istek:connect
    Gateway-->>İstemci: yanıt (tamam)
    Note right of Gateway: veya yanıt hatası + kapatma
    Note left of İstemci: yük=hello-ok<br>anlık görüntü: iletişim durumu + sistem durumu

    Gateway-->>İstemci: olay:presence
    Gateway-->>İstemci: olay:tick

    İstemci->>Gateway: istek:agent
    Gateway-->>İstemci: yanıt:agent<br>alındı {runId, status:"accepted"}
    Gateway-->>İstemci: olay:agent<br>(akış halinde)
    Gateway-->>İstemci: yanıt:agent<br>son {runId, status, summary}
```

## Kablo protokolü (özet)

- Aktarım: JSON yükleri içeren WebSocket metin çerçeveleri.
- İlk çerçeve `connect` **olmalıdır**.
- El sıkışmadan sonra:
  - İstekler: `{type:"req", id, method, params}` → `{type:"res", id, ok, payload|error}`
  - Olaylar: `{type:"event", event, payload, seq?, stateVersion?}`
- `hello-ok.features.methods` / `events`, çağrılabilir her yardımcı rotanın oluşturulmuş bir dökümü değil, keşif meta verileridir.
- Paylaşılan gizli anahtar kimlik doğrulaması, yapılandırılmış Gateway kimlik doğrulama moduna bağlı olarak `connect.params.auth.token` veya `connect.params.auth.password` kullanır.
- Tailscale Serve (`gateway.auth.allowTailscale: true`) veya geri döngü dışı `gateway.auth.mode: "trusted-proxy"` gibi kimlik taşıyan modlar, kimlik doğrulamayı `connect.params.auth.*` yerine istek başlıklarından karşılar.
- Özel giriş `gateway.auth.mode: "none"`, paylaşılan gizli anahtar kimlik doğrulamasını tamamen devre dışı bırakır; herkese açık/güvenilmeyen girişlerde bu modu kapalı tutun.
- Yan etkili yöntemlerin (`send`, `agent`) güvenle yeniden denenebilmesi için eşgüçlülük anahtarları gerekir; sunucu kısa ömürlü bir yinelenenleri ayıklama önbelleği tutar.
- Node'lar, `connect` içinde yetenekler/komutlar/izinlerle birlikte `role: "node"` içermelidir.

## Eşleştirme ve yerel güven

- Tüm WS istemcileri (operatörler + Node'lar), `connect` üzerinde bir **cihaz kimliği** içerir.
- Yeni cihaz kimlikleri eşleştirme onayı gerektirir; Gateway, sonraki bağlantılar için bir **cihaz belirteci** verir.
- Aynı ana bilgisayardaki kullanıcı deneyimini sorunsuz tutmak için doğrudan yerel geri döngü bağlantıları otomatik olarak onaylanabilir.
- OpenClaw ayrıca güvenilir paylaşılan gizli anahtar yardımcı akışları için dar kapsamlı bir arka uç/kapsayıcı içi kendi kendine bağlantı yoluna sahiptir.
- Aynı ana bilgisayardaki tailnet bağlamaları da dahil olmak üzere tailnet ve LAN bağlantıları yine açık eşleştirme onayı gerektirir.
- Tüm bağlantılar `connect.challenge` tek kullanımlık değerini imzalamalıdır. `v3` imza yükü ayrıca `platform` ve `deviceFamily` değerlerini de bağlar; Gateway, yeniden bağlantıda eşleştirilmiş meta verileri sabitler ve meta veri değişiklikleri için onarım eşleştirmesi gerektirir.
- **Yerel olmayan** bağlantılar yine açık onay gerektirir.
- Gateway kimlik doğrulaması (`gateway.auth.*`), yerel veya uzak fark etmeksizin **tüm** bağlantılara uygulanmaya devam eder.

Ayrıntılar: [Gateway protokolü](/tr/gateway/protocol), [Eşleştirme](/tr/channels/pairing),
[Güvenlik](/tr/gateway/security).

## Protokol türleri ve kod üretimi

- Protokolü TypeBox şemaları tanımlar.
- JSON Schema bu şemalardan oluşturulur.
- Swift modelleri JSON Schema'dan oluşturulur.

## Uzaktan erişim

- Tercih edilen: Tailscale veya VPN.
- Alternatif: SSH tüneli

  ```bash
  ssh -N -L 18789:127.0.0.1:18789 user@gateway-host
  ```

- Aynı el sıkışma ve kimlik doğrulama belirteci tünel üzerinden de geçerlidir.
- Uzak kurulumlarda WS için TLS ve isteğe bağlı sabitleme etkinleştirilebilir.

## İşletim anlık görüntüsü

- Başlatma: `openclaw gateway` (ön planda, günlükler standart çıktıya yazılır).
- Sistem durumu: WS üzerinden `health` (ayrıca `hello-ok` içinde bulunur).
- Gözetim: otomatik yeniden başlatma için launchd/systemd.

## Değişmezler

- Her ana bilgisayarda tek bir Baileys oturumunu tam olarak bir Gateway yönetir.
- El sıkışma zorunludur; JSON olmayan veya ilk çerçevesi connect olmayan bağlantılar doğrudan kapatılır.
- Olaylar yeniden oynatılmaz; istemciler boşluk oluştuğunda yenileme yapmalıdır.

## İlgili

- [Agent Döngüsü](/tr/concepts/agent-loop) — ayrıntılı agent yürütme döngüsü
- [Gateway Protokolü](/tr/gateway/protocol) — WebSocket protokol sözleşmesi
- [Kuyruk](/tr/concepts/queue) — komut kuyruğu ve eşzamanlılık
- [Güvenlik](/tr/gateway/security) — güven modeli ve sağlamlaştırma
