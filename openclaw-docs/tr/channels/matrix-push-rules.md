---
read_when:
    - Kendi barındırdığınız Synapse veya Tuwunel için Matrix sessiz akışını ayarlama
    - Kullanıcılar her önizleme düzenlemesinde değil, yalnızca bloklar tamamlandığında bildirim almak istiyor
summary: Sessiz tamamlanmış önizleme düzenlemeleri için alıcı başına Matrix anlık bildirim kuralları
title: Sessiz önizlemeler için Matrix push kuralları
x-i18n:
    generated_at: "2026-07-26T23:12:39Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1c58e7e796c3ae6d1ee25de229e4592ab8b4fb4d0d50a9cf868ab5ef35b1dab5
    source_path: channels/matrix-push-rules.md
    workflow: 16
---

`channels.matrix.streaming.mode`, `"quiet"` olduğunda OpenClaw, tek bir önizleme olayını yerinde düzenleyerek yanıtı akış halinde iletir. Önizlemeler, bildirim oluşturmayan `m.notice` olayları olarak gönderilir ve tamamlanan düzenleme `content["com.openclaw.finalized_preview"] = true` ile işaretlenir. Matrix istemcileri, yalnızca kullanıcıya özel bir anında bildirim kuralı işaretleyiciyle eşleşirse bu son düzenleme için bildirim gönderir. Bu sayfa, Matrix'i kendi sunucusunda barındıran ve her alıcı hesabı için bu kuralı yüklemek isteyen operatörlere yöneliktir.

`streaming.mode: "progress"`, taslaklarını aynı yol üzerinden tamamlar; dolayısıyla aynı kural, ilerleme modunda tamamlanan düzenlemeler için de tetiklenir.

Yalnızca standart Matrix bildirim davranışını istiyorsanız `streaming.mode: "partial"` kullanın veya akışı kapalı bırakın. Bkz. [Matrix kanal kurulumu](/tr/channels/matrix#streaming-previews).

## Ön koşullar

- alıcı kullanıcı = bildirimi alması gereken kişi
- bot kullanıcısı = yanıtı gönderen OpenClaw Matrix hesabı
- aşağıdaki API çağrıları için alıcı kullanıcının erişim belirtecini kullanın
- anında bildirim kuralındaki `sender` değerini bot kullanıcısının tam MXID'siyle eşleştirin
- alıcı hesabında çalışan anında bildirim göndericileri zaten bulunmalıdır; sessiz önizleme kuralları yalnızca normal Matrix anında bildirim teslimi sağlıklı olduğunda çalışır

## Adımlar

<Steps>
  <Step title="Sessiz önizlemeleri yapılandırın">

```json5
{
  channels: {
    matrix: {
      streaming: { mode: "quiet" },
    },
  },
}
```

  </Step>

  <Step title="Alıcının erişim belirtecini alın">
    Mümkün olduğunda mevcut bir istemci oturumu belirtecini yeniden kullanın. Yeni bir tane oluşturmak için:

```bash
curl -sS -X POST \
  "https://matrix.example.org/_matrix/client/v3/login" \
  -H "Content-Type: application/json" \
  --data '{
    "type": "m.login.password",
    "identifier": { "type": "m.id.user", "user": "@alice:example.org" },
    "password": "REDACTED"
  }'
```

  </Step>

  <Step title="Anında bildirim göndericilerinin mevcut olduğunu doğrulayın">

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushers"
```

Hiçbir anında bildirim göndericisi dönmezse devam etmeden önce bu hesap için normal Matrix anında bildirim teslimini düzeltin.

  </Step>

  <Step title="Geçersiz kılan anında bildirim kuralını yükleyin">
    Tamamlanan önizleme işaretleyicisiyle ve gönderen olarak bot MXID'siyle eşleşen bir kural yükleyin:

```bash
curl -sS -X PUT \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname" \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  --data '{
    "conditions": [
      { "kind": "event_match", "key": "type", "pattern": "m.room.message" },
      {
        "kind": "event_property_is",
        "key": "content.m\\.relates_to.rel_type",
        "value": "m.replace"
      },
      {
        "kind": "event_property_is",
        "key": "content.com\\.openclaw\\.finalized_preview",
        "value": true
      },
      { "kind": "event_match", "key": "sender", "pattern": "@bot:example.org" }
    ],
    "actions": [
      "notify",
      { "set_tweak": "sound", "value": "default" },
      { "set_tweak": "highlight", "value": false }
    ]
  }'
```

    Çalıştırmadan önce şunları değiştirin:

    - `https://matrix.example.org`: ana sunucunuzun temel URL'si
    - `$USER_ACCESS_TOKEN`: alıcı kullanıcının erişim belirteci
    - `openclaw-finalized-preview-botname`: alıcı başına her bot için benzersiz bir kural kimliği (kalıp: `openclaw-finalized-preview-<botname>`)
    - `@bot:example.org`: alıcının değil, OpenClaw botunuzun MXID'si

  </Step>

  <Step title="Doğrulayın">

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname"
```

Ardından akış halinde iletilen bir yanıtı test edin. Sessiz modda oda, sessiz bir taslak önizlemesi gösterir ve blok veya tur tamamlandığında bildirim gönderir.

  </Step>
</Steps>

Kuralı daha sonra kaldırmak için alıcının belirteciyle aynı kural URL'sine `DELETE` isteği gönderin.

## Çoklu bot notları

Anında bildirim kuralları `ruleId` ile anahtarlanır: aynı kimlik için `PUT` çağrısını yeniden çalıştırmak tek bir kuralı günceller. Aynı alıcıya bildirim gönderen birden fazla OpenClaw botu için, her bot adına farklı bir gönderen eşleşmesine sahip bir kural oluşturun.

Kullanıcı tanımlı yeni `override` kuralları, sunucunun varsayılan engelleme kurallarının önüne eklenir; dolayısıyla ek bir sıralama parametresi gerekmez. Kural yalnızca yerinde tamamlanabilen salt metin önizleme düzenlemelerini etkiler; medya yanıtları, geçerliliğini yitirmiş önizleme için geri dönüşler ve Matrix bahsetmelerini etkinleştirecek son metinler bunun yerine normal bildirim oluşturan mesajlar olarak teslim edilir.

## Ana sunucu notları

<AccordionGroup>
  <Accordion title="Synapse">
    Özel bir `homeserver.yaml` değişikliği gerekmez. Normal Matrix bildirimleri bu kullanıcıya zaten ulaşıyorsa alıcı belirteci ve yukarıdaki `pushrules` çağrısı ana kurulum adımıdır.

    Synapse'i bir ters proxy veya worker'ların arkasında çalıştırıyorsanız `/_matrix/client/.../pushrules/` isteğinin Synapse'e doğru şekilde ulaştığından emin olun. Anında bildirim teslimi ana süreç veya `synapse.app.pusher` / yapılandırılmış anında bildirim worker'ları tarafından gerçekleştirilir; bunların sağlıklı olduğundan emin olun.

    Kural, 2023'te Synapse'e eklenen `event_property_is` anında bildirim kuralı koşulunu (MSC3758, anında bildirim kuralı v1.10) kullanır. Eski Synapse sürümleri `PUT pushrules/...` çağrısını kabul eder ancak koşulu hiçbir zaman eşleştirmeden sessizce geçer; tamamlanan bir önizleme düzenlemesinde bildirim gelmezse Synapse'i yükseltin.

  </Accordion>

  <Accordion title="Tuwunel">
    Synapse ile aynı akış geçerlidir; tamamlanan önizleme işaretleyicisi için Tuwunel'e özgü bir yapılandırma gerekmez.

    Kullanıcı başka bir cihazda etkin durumdayken bildirimler kayboluyorsa `suppress_push_when_active` seçeneğinin etkin olup olmadığını kontrol edin. Tuwunel bu seçeneği 1.4.2 (Eylül 2025) sürümünde ekledi ve bu seçenek, bir cihaz etkinken diğer cihazlara gönderilen anında bildirimleri kasıtlı olarak engelleyebilir.

  </Accordion>
</AccordionGroup>

## İlgili

- [Matrix kanal kurulumu](/tr/channels/matrix)
- [Akış kavramları](/tr/concepts/streaming)
