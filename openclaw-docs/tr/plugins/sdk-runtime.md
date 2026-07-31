---
read_when:
    - Bir Plugin içinden temel yardımcıları çağırmanız gerekiyor (TTS, STT, görüntü oluşturma, web araması, Gateway, alt ajan, Node'lar)
    - api.runtime'ın neleri kullanıma sunduğunu anlamak istiyorsunuz
    - Plugin kodundan yapılandırma, aracı veya medya yardımcılarına erişiyorsunuz
sidebarTitle: Runtime helpers
summary: api.runtime -- pluginlerin kullanabildiği enjekte edilmiş çalışma zamanı yardımcıları
title: Plugin çalışma zamanı yardımcıları
x-i18n:
    generated_at: "2026-07-26T23:29:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ff1d901de8ec70011eeaafbab7b3cc30709fc95894c7ba4f4346c026de682cd0
    source_path: plugins/sdk-runtime.md
    workflow: 16
---

Kayıt sırasında her plugine enjekte edilen `api.runtime` nesnesi için başvuru. Ana bilgisayarın iç bileşenlerini doğrudan içe aktarmak yerine bu yardımcıları kullanın.

<CardGroup cols={2}>
  <Card title="Kanal pluginleri" href="/tr/plugins/sdk-channel-plugins">
    Kanal pluginleri için bu yardımcıları bağlam içinde kullanan adım adım kılavuz.
  </Card>
  <Card title="Sağlayıcı pluginleri" href="/tr/plugins/sdk-provider-plugins">
    Sağlayıcı pluginleri için bu yardımcıları bağlam içinde kullanan adım adım kılavuz.
  </Card>
</CardGroup>

```typescript
register(api) {
  const runtime = api.runtime;
}
```

`api.runtime.version`, paylaşılan sürüm çözümleyicisinden alınan güncel OpenClaw ürün sürümüdür; böylece pluginler CLI'ın bildirdiği değerle aynı değeri görür.

## Yapılandırma yükleme ve yazma

Etkin çağrı yoluna önceden geçirilmiş yapılandırmayı tercih edin; örneğin kayıt sırasında `api.config` veya kanal/sağlayıcı geri çağırmalarındaki bir `cfg` bağımsız değişkeni. Bu, yoğun kullanılan yollarda yapılandırmayı yeniden ayrıştırmak yerine tek bir işlem anlık görüntüsünün iş boyunca aktarılmasını sağlar.

`api.runtime.config.current()` yalnızca uzun ömürlü bir işleyicinin güncel işlem anlık görüntüsüne ihtiyaç duyduğu ve bu işleve herhangi bir yapılandırma geçirilmediği durumlarda kullanın. Döndürülen değer salt okunurdur; düzenlemeden önce kopyalayın veya bir mutasyon yardımcısı kullanın.

Araç fabrikaları `ctx.runtimeConfig` ile birlikte `ctx.getRuntimeConfig()` alır. Araç tanımı oluşturulduktan sonra yapılandırma değişebiliyorsa uzun ömürlü bir aracın `execute` geri çağırması içinde alıcıyı kullanın.

Değişiklikleri `api.runtime.config.mutateConfigFile(...)` veya `api.runtime.config.replaceConfigFile(...)` ile kalıcılaştırın. Her yazma işlemi açık bir `afterWrite` politikası seçmelidir:

- `afterWrite: { mode: "auto" }`, gateway yeniden yükleme planlayıcısının karar vermesine izin verir.
- `afterWrite: { mode: "restart", reason: "..." }`, yazan taraf çalışırken yeniden yüklemenin güvenli olmadığını bildiğinde temiz bir yeniden başlatmayı zorunlu kılar.
- `afterWrite: { mode: "none", reason: "..." }`, yalnızca çağıran taraf sonraki işlemi üstlendiğinde otomatik yeniden yüklemeyi/yeniden başlatmayı engeller.

Mutasyon yardımcıları `afterWrite` ile birlikte türü belirlenmiş bir `followUp` özeti döndürür; böylece çağıranlar yeniden başlatma isteyip istemediklerini günlüğe kaydedebilir veya test edebilir. Yeniden başlatmanın gerçekte ne zaman gerçekleşeceği yine gateway'in sorumluluğundadır.

Çalışma zamanı yapılandırmasına erişmek ve yazmak için `current()`, geçirilmiş bir `cfg`, `mutateConfigFile(...)` veya
`replaceConfigFile(...)` kullanın.

Doğrudan SDK içe aktarımlarında geniş `openclaw/plugin-sdk/config-runtime` uyumluluk varili yerine odaklanmış yapılandırma alt yollarını tercih edin: türler için `config-contracts`, güncel işlem anlık görüntüleri için `runtime-config-snapshot` ve yazma işlemleri için `config-mutation`. Giriş kapsamlı değerleri `api.pluginConfig` üzerinden okuyun; sağlanan bir araç bağlamını yalnızca çalışma zamanı genelindeki yapılandırma anlık görüntüsü için kullanın ve plugine özgü birleştirmeyi bu sınırda tutun. Paketlenmiş plugin testleri, geniş uyumluluk varilini taklit etmek yerine bu odaklanmış alt yolları doğrudan taklit etmelidir.

Dahili OpenClaw çalışma zamanı kodu da aynı yaklaşımı izler: yapılandırmayı CLI, gateway veya işlem sınırında bir kez yükleyip ardından bu değeri aktarır. Başarılı mutasyon yazmaları işlem çalışma zamanı anlık görüntüsünü yeniler ve dahili revizyonunu ilerletir; uzun ömürlü önbellekler yapılandırmayı yerel olarak serileştirmek yerine çalışma zamanının sahip olduğu önbellek anahtarını temel almalıdır. Uzun ömürlü çalışma zamanı modüllerinde ortamdan yapılan `loadConfig()` çağrıları için sıfır toleranslı bir tarayıcı bulunur; geçirilmiş bir `cfg`, istek `context.getRuntimeConfig()` veya açık bir işlem sınırında `getRuntimeConfig()` kullanın.

Sağlayıcı ve kanal yürütme yolları, yapılandırmayı geri okuma veya düzenleme amacıyla döndürülen bir dosya anlık görüntüsünü değil, etkin çalışma zamanı yapılandırma anlık görüntüsünü kullanmalıdır. Dosya anlık görüntüleri, kullanıcı arayüzü ve yazma işlemleri için SecretRef işaretçileri gibi kaynak değerleri korur; sağlayıcı geri çağırmaları ise çözümlenmiş çalışma zamanı görünümüne ihtiyaç duyar. Bir yardımcı etkin kaynak anlık görüntüsüyle veya etkin çalışma zamanı anlık görüntüsüyle çağrılabiliyorsa kimlik bilgilerini okumadan önce `selectApplicableRuntimeConfig()` üzerinden yönlendirin.

## Yeniden kullanılabilir çalışma zamanı yardımcı programları

Bot tarafından oluşturulan gelen iletiler için gelen `botLoopProtection` olgularını kullanın. Çekirdek, politikayı tek bir kanala bağlamadan oturum kaydı ve dağıtımdan önce paylaşılan bellek içi kayan pencere korumasını uygular. Koruma `(scopeId, conversationId, participant pair)` anahtarlarını izler, bir çiftin her iki yönünü birlikte sayar, pencere bütçesi aşıldığında bekleme süresi uygular ve etkin olmayan girdileri fırsat buldukça temizler.

Bu davranışı operatörlere sunan kanal pluginleri, temel bütçeler için paylaşılan `channels.defaults.botLoopProtection` biçimini tercih etmeli ve ardından kanala/sağlayıcıya özgü geçersiz kılmaları bunun üzerine uygulamalıdır. Paylaşılan yapılandırma kullanıcıya yönelik olduğundan saniye kullanır:

```typescript
type ChannelBotLoopProtectionConfig = {
  enabled?: boolean;
  maxEventsPerWindow?: number;
  windowSeconds?: number;
  cooldownSeconds?: number;
};
```

Normalleştirilmiş bot çifti olgularını çözümlenmiş turla birlikte geçirin. Çekirdek varsayılanları, birim dönüşümünü ve `enabled` semantiğini çözümler:

```typescript
return {
  channel: "example",
  routeSessionKey,
  storePath,
  ctxPayload,
  recordInboundSession,
  runDispatch,
  botLoopProtection: {
    scopeId: "account-1",
    conversationId: "channel-1",
    senderId: "bot-a",
    receiverId: "bot-b",
    config: channelConfig.botLoopProtection,
    defaultsConfig: runtimeConfig.channels?.defaults?.botLoopProtection,
    defaultEnabled: allowBotsMode !== "off",
  },
};
```

`openclaw/plugin-sdk/pair-loop-guard-runtime` öğesini yalnızca paylaşılan gelen yanıt çalıştırıcısından geçmeyen özel
iki taraflı olay döngüleri için doğrudan kullanın.

## Çalışma zamanı ad alanları

<AccordionGroup>
  <Accordion title="api.runtime.agent">
    Agent kimliği, dizinleri ve oturum yönetimi.

    ```typescript
    // Agent'ın çalışma dizinini çözümle (agentId gereklidir)
    const agentDir = api.runtime.agent.resolveAgentDir(cfg, agentId);

    // Agent çalışma alanını çözümle
    const workspaceDir = api.runtime.agent.resolveAgentWorkspaceDir(cfg, agentId);

    // Agent kimliğini al
    const identity = api.runtime.agent.resolveAgentIdentity(cfg);

    // Varsayılan düşünme düzeyini al
    const thinking = api.runtime.agent.resolveThinkingDefault({
      cfg,
      provider,
      model,
    });

    // Kullanıcı tarafından sağlanan düşünme düzeyini etkin sağlayıcı profiline göre doğrula
    const policy = api.runtime.agent.resolveThinkingPolicy({ provider, model });
    const level = api.runtime.agent.normalizeThinkingLevel("extra high");
    if (level && policy.levels.some((entry) => entry.id === level)) {
      // Düzeyi gömülü bir çalıştırmaya geçir
    }

    // Agent zaman aşımını al
    const timeoutMs = api.runtime.agent.resolveAgentTimeoutMs(cfg);

    // Çalışma alanının var olduğundan emin ol
    await api.runtime.agent.ensureAgentWorkspace(cfg);

    // Gömülü bir agent turu çalıştır
    const result = await api.runtime.agent.runEmbeddedAgent({
      sessionId: "my-plugin:task-1",
      runId: crypto.randomUUID(),
      workspaceDir: api.runtime.agent.resolveAgentWorkspaceDir(cfg, agentId),
      prompt: "En son değişiklikleri özetle",
      timeoutMs: api.runtime.agent.resolveAgentTimeoutMs(cfg),
    });
    ```

    `runEmbeddedAgent(...)`, plugin kodundan normal bir OpenClaw agent turu başlatmak için kullanılan tarafsız yardımcıdır. Kanal tarafından tetiklenen yanıtlarla aynı sağlayıcı/model çözümlemesini ve agent çalıştırma düzeneği seçimini kullanır.

    `runEmbeddedPiAgent(...)`, mevcut pluginler için kullanımdan kaldırılmış bir uyumluluk diğer adı olarak kalır. Yeni kod `runEmbeddedAgent(...)` kullanmalıdır.

    `resolveCliBackendDispatchEligibility({ provider, model, agentId, authProfileId, config, agentDir, workspaceDir })`, gömülü çalıştırıcıya ait CLI arka uç dağıtım kararını (rota, arka ucun bildirdiği `subscriptionAuthDispatch` yeteneği, saklanan kimlik bilgisi modu — açıkça sabitlenmiş bir `authProfileId` değerine uyarak) gömülü çalıştırmaları `cliBackendDispatch: "subscription-auth"` kapsamına dahil eden çağıranlarla paylaşır. Çalıştırma CLI arka ucu üzerinden yürütülecekse `{ provider }`, doğrudan geçişte kalacaksa `undefined` döndürür; böylece çağıranlar gerçekten yürütülecek çalıştırma için zaman aşımı bütçesi ayırabilir.

    `resolveThinkingPolicy(...)`, sağlayıcının/modelin desteklediği düşünme düzeylerini ve isteğe bağlı varsayılanı döndürür. Sağlayıcı pluginleri, düşünme kancaları aracılığıyla modele özgü profile sahip olduğundan araç pluginleri sağlayıcı listelerini içe aktarmak veya çoğaltmak yerine bu çalışma zamanı yardımcısını çağırmalıdır.

    `normalizeThinkingLevel(...)`, `on`, `x-high` veya `extra high` gibi kullanıcı metinlerini çözümlenmiş politikaya göre denetlemeden önce kurallı saklama düzeyine dönüştürür.

    **Oturum deposu yardımcıları** `api.runtime.agent.session` altındadır:

    ```typescript
    const entry = api.runtime.agent.session.getSessionEntry({ agentId, sessionKey });
    for (const { sessionKey, entry } of api.runtime.agent.session.listSessionEntries({ agentId })) {
      // Eski sessions.json biçimine bağımlı olmadan oturum satırları üzerinde yineleme yap.
    }
    await api.runtime.agent.session.patchSessionEntry({
      agentId,
      sessionKey,
      update: (entry) => ({ thinkingLevel: "high" }),
    });

    const created = await api.runtime.agent.session.createSessionEntry({
      cfg,
      key: "agent:main:my-plugin:task-1",
      initialEntry: {
        agentHarnessId: "my-harness",
        modelSelectionLocked: true,
        pluginExtensions: { "my-plugin": { phase: "initializing" } },
      },
      afterCreate: async () => ({
        pluginExtensions: { "my-plugin": { phase: "ready" } },
      }),
    });

    const storePath = api.runtime.agent.session.resolveStorePath(cfg.session?.store, { agentId });
    await api.runtime.agent.session.runWithWorkAdmission(
      { storePath, sessionKey },
      async (signal) => {
        // Oturumu oluştur veya güncelle, ardından signal değerini kabul edilen agent çalıştırmasına geçir.
      },
    );
    ```

    Oturum iş akışları için `getSessionEntry(...)`, `listSessionEntries(...)`, `patchSessionEntry(...)` veya `upsertSessionEntry(...)` tercih edin. Bu yardımcılar oturumları agent/oturum kimliğiyle adresler; böylece pluginler eski `sessions.json` depolama biçimine bağımlı olmaz. Oturum etkinliğini yenilememesi gereken yalnızca meta veri yamaları için `preserveActivity: true`, yalnızca geri çağırma eksiksiz bir girdi döndürdüğünde ve silinen alanların silinmiş kalması gerektiğinde `replaceEntry: true` kullanın. Doctor ve geçiş yolları, kurallı depoda tek bir atomik onarım için `fallbackEntry`, `skipMaintenance` ve `requireWriteSuccess` öğelerini birleştirebilir.

    `createSessionEntry(...)`, yeni bir kurallı oturum satırı ve transkript oluşturur. Güvenilir `initialEntry` yüzeyi bilinçli olarak dardır: boş olmayan bir `agentHarnessId`, isteğe bağlı `modelSelectionLocked: true` ve isteğe bağlı `pluginExtensions`. Enjekte edilen çalışma zamanı, `registerAgentHarness(...)` aracılığıyla yalnızca çağıran pluginin sahip olduğu çalıştırma düzeneği kimliklerini kabul eder; bu, işlem içi pluginler arasında bir korumalı alan değil, sahiplik değişmezidir. Mevcut bir satırı reddeder; `label` ve `spawnedCwd`, güvenilir girdi yamaları yerine ayrı oluşturma alanlarıdır.

    Oluşturma işlemi, oturum yaşam döngüsü mutasyon engelini `afterCreate` boyunca tutar; böylece yeni işler plugine ait başlatma işleminin tamamlanmasını bekler ve önceden kabul edilmiş işler oluşturmanın başarısız olmasına neden olur. Geri çağırma, oluşturulan durumun bir kopyasını alır. Bir yama döndürürse bu yama yalnızca `pluginExtensions` içerebilir ve değeri eksiksiz nihai `pluginExtensions` alanıdır. Geri çağırma veya nihai kalıcılaştırma hatası, değiştirilmemiş yeni satırı ve transkripti geri alır; korumalı geri alma, eşzamanlı olarak değiştirilmiş veya sahiplenilmiş bir satırı korur. `recoverMatchingInitialEntry: true` yalnızca kalıcılaştırılmış güvenilir alanlar tam olarak eşleştiğinde kesintiye uğramış başlatmayı yeniden denemek içindir ve kurtarma işlemi `afterCreate` öğesinin nihai bir yama döndürmesini gerektirir.

    Bir plugin kalıcılaştırılmış bir oturum üzerinde çalışma başlattığında `runWithWorkAdmission(...)` kullanın. Geri çağırma arşivlenmiş veya eşzamanlı olarak değiştirilmiş oturumları reddeder, arşivleme/sıfırlama/silme mutasyonlarını tamamlanana kadar eşgüdümlü tutar ve agent çalıştırmasına iletilmesi gereken bir `AbortSignal` alır. Bir çalıştırma düzeneği, deneysel `delegatedExecutionPluginIds` kayıt alanı aracılığıyla güvenilir yürütme temsilcilerini açıkça adlandırabilir. Temsilciler yalnızca tam olarak eşleşen, mevcut ve modeli kilitlenmiş bir oturumu kabul edip çalıştırabilir; tüm oturum mutasyonları çalıştırma düzeneği sahibiyle sınırlı kalır. Bkz. [Agent çalıştırma düzeneği pluginleri](/tr/plugins/sdk-agent-harness#delegated-execution).

    Bakım ve onarım pluginleri, kapsamı belirlenmiş tek bir oturum girdisi için `deleteSessionEntry(...)`, yaşam döngüsü tarafından yönetilen geçici çalışma oturumları için `cleanupSessionLifecycleArtifacts(...)` ve bir depoyu değiştirmeden önce `resolveSessionStoreBackupPaths(...)` kullanabilir. Silme işleminin eşzamanlı bir oturum güncellemesiyle yarışmaması gerektiğinde `expectedSessionId` ve `expectedUpdatedAt` değerlerini iletin; önceki anlık görüntüde oturum kimliği yoksa `expectedSessionId: null` kullanın. Bu yardımcılar, genel bir depo silme API'si değil, dar kapsamlı onarım/yaşam döngüsü yüzeyleridir.

    `resolveStorePath(...)` ve `updateSessionStoreEntry(...)` oturum yardımcılarını tamamlar: `resolveStorePath`, belirli bir kapsam için oturum deposu yolunu çözümler; `updateSessionStoreEntry({ storePath, sessionKey, update })` ise çağıran bu yolu zaten biliyorsa tek bir girdiyi doğrudan depo yoluna göre yamalar.

    `loadTranscriptEventsSync(...)`, eşzamansız transkript çalışma zamanını kullanamayan eşzamanlı doctor ve onarım yolları için kullanılabilir. Ham `SessionStoreTranscriptEvent` kayıtlarını döndürür. Normal plugin çalışma zamanı kodu `openclaw/plugin-sdk/session-transcript-runtime` tercih etmelidir.

    `formatSqliteSessionFileMarker(...)`, `parseSqliteSessionFileMarker(...)` ve `sqliteSessionFileMarkerMatchesSession(...)`, hâlâ `sessionFile` adlı eski bir alanı alan kodlar için geçiş yardımcılarıdır. Ayrıştırılmış bir SQLite işaretçisi, etkin bir SQLite transkript hedefini tanımlar; bir dosya sistemi yolu değildir. Yeni API'ler işaretçi dizeleri yerine türü belirlenmiş oturum kimliği taşımalıdır.

    Transkript okuma ve yazma işlemleri için `openclaw/plugin-sdk/session-transcript-runtime` içe aktarın ve `{ agentId, sessionKey, sessionId }` ile birlikte `resolveSessionTranscriptIdentity(...)`, `resolveSessionTranscriptTarget(...)`, `readSessionTranscriptEvents(...)`, `readSessionTranscriptRawDelta(...)`, `readSessionTranscriptVisibleMessageDelta(...)`, `readVisibleSessionTranscriptMessageEntries(...)`, `appendSessionTranscriptMessageByIdentity(...)`, `publishSessionTranscriptUpdateByIdentity(...)` veya `withSessionTranscriptWriteLock(...)` kullanın. Bu API'ler, pluginlerin etkin transkript dosya yollarına bağımlı olmadan bir transkripti tanımlamasına, ham olayları veya dal açısından güvenli görünür ileti girdilerini okumasına, iletiler eklemesine, güncellemeler yayımlamasına ve ilgili işlemleri aynı transkript yazma kilidi altında çalıştırmasına olanak tanır. `readVisibleSessionTranscriptMessageEntries(...)` sıralı okuma meta verilerini döndürür; `seq` alanı sürdürülebilir bir imleç değildir.

    `appendSessionTranscriptMessageByIdentity(...)`, zaten standartlaştırılmış bir iletinin düşük seviyeli ekleme işlemidir. Pluginler, üst düzey `MediaPath`, `MediaPaths`, `MediaUrl`, `MediaUrls`, `MediaType` veya `MediaTypes` içeren medya taşıyan kullanıcı satırları oluşturmamalıdır. Kanal girişi, sıralı olguları `MsgContext.media` üzerinden iletmeli ve kullanıcı turunun kalıcılaştırılmasının yönetimini ana sisteme bırakmalıdır. Ana sistem tarafından hazırlanmış, kalıcılaştırılan bir kullanıcı iletisi, standartlaştırılmış sıralı olguları `message.__openclaw.media` altında taşır; genel ekleme API'si eski paralel dizileri çıkarımlamaz veya onarmaz.

    `readSessionTranscriptRawDelta(...)`, sınırlandırılmış bir `page`, `reset` veya `missing` sonucu döndürür. Opak `page.cursor` değerini sonraki çağrıya iletin. Yalnızca ekleme işlemleri imleci korurken transkriptin değiştirilmesi, yeni bir önyükleme imleciyle `reset` döndürür. Sayfalar varsayılan olarak 1.000 olay ve 1.000.000 serileştirilmiş bayt içerir; çağıranlar en fazla 10.000 olay ve 64 MiB isteyebilir. Yalnızca bir sonraki olay `maxBytes` değerini aştığında sayfa boş olur ve `requiredBytes` bildirir; değer 64 MiB'den büyük değilse en az bu bayt sınırıyla yeniden deneyin. Daha büyük tekil olaylar tam okuma API'sini gerektirir. Bir imleç yalnızca konumu tanımlar ve başka bir oturuma hiçbir zaman erişim sağlamaz.

    `readSessionTranscriptVisibleMessageDelta(...)`, ana sistem tarafından yönetilen etkin ileti izdüşümü üzerinde aynı sınırlandırılmış önyükleme ve sürdürme biçimini sağlar. İletileri en eskiden en yeniye doğru döndürür; böylece bağlam motorları ilk geçmişi tüketebilir ve opak imleci filigranları olarak kalıcılaştırabilir. İmleci değiştirmeden saklayın ve döndürün; bu bir yetkilendirme kimlik bilgisi değil, sürdürme ipucudur. Doğrusal eklemeler, son döndürülen iletiden sonra sürdürülür. Transkriptin değiştirilmesi, sabitleyicisi etkin daldan ayrılmış veya etkin dal içinde taşınmış bir imleç, hatalı biçimlendirilmiş imleçler ve oturumlar arası imleçler, yeni bir önyükleme imleciyle `reset` döndürür. Sayı ve bayt varsayılanları ile üst sınırları, ham delta API'siyle aynıdır. Bir dal değişikliğinden sonra etkin izdüşüm yeniden oluşturulurken sonuç, `projection_rebuilding` nedeniyle `unavailable` olur; etkin bir transkript dosyasına geri dönmek yerine daha sonra yeniden deneyin.

    Eski tam depo ve etkin transkript dosyası yardımcıları artık plugin SDK'sından dışa aktarılmamaktadır. Oturum meta verileri için kapsamlı girdi yardımcılarını, etkin transkript işlemleri içinse transkript kimliği yardımcılarını kullanın. Dosya yapıtlarına ihtiyaç duyan arşiv/destek iş akışları, etkin oturum çalışma zamanı API'leri yerine kendilerine ayrılmış arşiv yüzeylerini kullanmalıdır.

  </Accordion>
  <Accordion title="api.runtime.agent.defaults">
    Varsayılan model ve sağlayıcı sabitleri:

    ```typescript
    const model = api.runtime.agent.defaults.model; // ör. "gpt-5.6-sol"
    const provider = api.runtime.agent.defaults.provider; // ör. "openai"
    ```

  </Accordion>

  <Accordion title="api.runtime.llm">
    Sağlayıcının iç bileşenlerini içe aktarmadan veya OpenClaw model/kimlik doğrulama/temel URL hazırlığını
    yinelemeden ana sistem tarafından yönetilen bir metin tamamlama işlemi çalıştırın.

    ```typescript
    const result = await api.runtime.llm.complete({
      messages: [{ role: "user", content: "Bu transkripti özetle." }],
      purpose: "my-plugin.summary",
      maxTokens: 512,
      temperature: 0.2,
      reasoning: "high",
    });
    ```

    Sağlayıcı düzenlemesi, bir HTTP isteği göndermeden önce yapılandırılmış yerel hizmetin
    yaşam döngüsünü de edinebilir:

    ```typescript
    const lease = await api.runtime.llm.acquireLocalService(
      {
        providerId,
        baseUrl,
        headers,
      },
      signal,
    );
    try {
      // Sağlayıcı isteğini gönderin ve tamamen tüketin.
    } finally {
      await lease?.release();
    }
    ```

    `acquireLocalService(...)`, kararlı ve genel bir sağlayıcı hizmeti SDK
    sözleşmesidir. Ana sistem, işlem yapılandırmasını
    `models.providers.<providerId>.localService` üzerinden çözümler; çağıranlar bir
    komut, bağımsız değişkenler, ortam veya yaşam döngüsü politikası sağlayamaz. İşlem başlatma,
    hazır olma durumu, tanılama ve boşta durdurma politikası ana sistemin iç sorumluluğunda kalır.

    Tam olarak yapılandırılmış sağlayıcı kimliğini ve çözümlenmiş istek temel URL'sini iletin.
    Takma adları bir bağdaştırıcı kimliğiyle değiştirmeyin: ayrı takma adlar ayrı
    yerel GPU ana sistemlerini gösterebilir. Ana sistem, Ollama ve LM
    Studio bağdaştırıcılarının kullandığı `/v1` normalleştirmesi dışında, yapılandırılmış
    sağlayıcı temel URL'siyle eşleşmeyen uç noktaları reddeder. Başlatma serileştirmesi, hazır olma yoklamaları,
    istek kiralamaları, iptal işlemleri ve boşta kapatma ana sistem tarafından yönetilir.

    Yardımcı, OpenClaw'ın yerleşik çalışma zamanıyla aynı basit tamamlama hazırlama
    yolunu ve ana sistem tarafından yönetilen çalışma zamanı yapılandırma anlık görüntüsünü kullanır. Bağlam motorları
    oturuma bağlı bir `llm.complete` yeteneği alır; böylece model çağrıları etkin
    oturumun aracısını kullanır ve sessizce varsayılan aracıya geri dönmez. Sonuç,
    mevcut olduğunda sağlayıcı/model/aracı ilişkilendirmesinin yanı sıra normalleştirilmiş token,
    önbellek ve tahmini maliyet kullanımını içerir.

    Seçili model için bir akıl yürütme düzeyi istemek üzere `reasoning` ayarlayın.
    Ana sistem, tamamlama işlemini göndermeden önce seçili sağlayıcı ve model için
    standart düşünme düzeylerini (`off`, `minimal`, `low`,
    `medium`, `high`, `xhigh`, `adaptive`, `max` ve `ultra`) normalleştirir. `adaptive`,
    `medium` olur; `max` ve `ultra`, destekleniyorsa `max`, aksi takdirde `xhigh` olur.

    <Warning>
    Model geçersiz kılmaları, yapılandırmada `plugins.entries.<id>.llm.allowModelOverride: true` üzerinden operatörün açık onayını gerektirir. Güvenilir pluginleri belirli standart `provider/model` hedefleriyle sınırlandırmak için `plugins.entries.<id>.llm.allowedModels` kullanın. Aracılar arası tamamlamalar `plugins.entries.<id>.llm.allowAgentIdOverride: true` gerektirir.
    </Warning>

  </Accordion>
  <Accordion title="api.runtime.gateway">
    Geçerli pluginin güvenilir çalışma zamanı kimliğini koruyarak süreç içinde başka bir Gateway yöntemini
    çağırın. Bu, geri döngü WebSocket bağlantısı açmadan plugin tarafından yönetilen
    Gateway yeteneklerini birleştiren paketlenmiş veya güvenilir resmî pluginler için tasarlanmıştır.

    ```typescript
    if (await api.runtime.gateway.isAvailable()) {
      const result = await api.runtime.gateway.request<{ callId: string }>(
        "voicecall.start",
        { to: "+15550001234", mode: "conversation" },
        { timeoutMs: 60_000 },
      );
    }
    ```

    İstekler `operator.write` kapsamını kullanır ve yönetici kapsamı vermez. Rastgele haricî
    pluginlerden gelen çağrılar reddedilir. Başarısız yöntemler, yapılandırılmış
    `details`, yeniden deneme meta verilerini ve kurtarma akışları için Gateway hata kodunu koruyan bir `GatewayClientRequestError` oluşturur. Bağımsız aracı süreçlerinde de çalışabilen araçlardan bu yolu seçmeden önce `isAvailable()`
    kullanın.

  </Accordion>
  <Accordion title="api.runtime.subagent">
    Arka plan alt aracı çalıştırmalarını başlatın ve yönetin.

    ```typescript
    // Bir alt aracı çalıştırması başlatın
    const { runId } = await api.runtime.subagent.run({
      sessionKey: "agent:main:subagent:search-helper",
      message: "Bu sorguyu odaklanmış takip aramalarına genişlet.",
      toolsAlsoAllow: ["my_plugin_progress"],
      provider: "openai", // isteğe bağlı geçersiz kılma
      model: "gpt-5.6-sol", // isteğe bağlı geçersiz kılma
      deliver: false,
    });

    // Tamamlanmasını bekleyin
    const result = await api.runtime.subagent.waitForRun({ runId, timeoutMs: 30000 });

    // Oturum iletilerini okuyun
    const { messages } = await api.runtime.subagent.getSessionMessages({
      sessionKey: "agent:main:subagent:search-helper",
      limit: 10,
    });

    // Bir oturumu silin
    await api.runtime.subagent.deleteSession({
      sessionKey: "agent:main:subagent:search-helper",
    });
    ```

    <Warning>
    Model geçersiz kılmaları (`provider`/`model`), yapılandırmada `plugins.entries.<id>.subagent.allowModelOverride: true` üzerinden operatörün açık onayını gerektirir. Güvenilmeyen pluginler yine de alt aracılar çalıştırabilir ancak geçersiz kılma istekleri reddedilir.
    </Warning>

    `toolsAlsoAllow`, çağıran plugin tarafından kaydedilmiş, tam olarak eşleşen ve benzersiz şekilde sahip olunan araçları çalışanın normal araç yüzeyine ekler. Çalışma zamanı, temel araçları ve başka bir pluginle paylaşılan adları reddeder. Açık izin listeleri ve retler dâhil olmak üzere profiller ve operatör araç politikaları uygulanmaya devam eder.

    `deleteSession(...)`, aynı plugin tarafından `api.runtime.subagent.run(...)` üzerinden oluşturulan oturumları silebilir. Rastgele kullanıcı veya operatör oturumlarının silinmesi yine de yönetici kapsamlı bir Gateway isteği gerektirir.

  </Accordion>
  <Accordion title="api.runtime.sandbox">
    Bir aracı oturumu için geçerli korumalı alan çalışma alanı yetkisini inceleyin.

    ```typescript
    const authority = api.runtime.sandbox.resolveWorkspaceAuthority({
      config: cfg,
      agentId,
      sessionKey,
    });

    const liveAuthority = await api.runtime.sandbox.prepareWorkspaceAuthority({
      config: cfg,
      agentId,
      sessionKey,
      workspaceDir,
      confinedToolNames: ["my_plugin_safe_tool"],
    });
    ```

    Sonuç, bu oturumun korumalı alanda olup olmadığını, çalışma alanının
    kullanılamaz, salt okunur veya yazılabilir olup olmadığını ve geçerli Docker, araç, oturum, tarayıcı
    veya yükseltilmiş politikanın bu çalışma alanından kaçabilmesi durumunda isteğe bağlı bir `confinementError`
    bildirir. Bunu, bir çalışana çağıranından daha fazla yetki vermemesi gereken
    ana sistem tarafından yönetilen yetkilendirme kararları için kullanın. Bu bir doğrulama
    yardımcısıdır; çağıranın kendi yetkilendirmesini denetlemenin yerine geçmez.

    `prepareWorkspaceAuthority(...)`, aynı politika denetimini gerçekleştirir ve ayrıca
    `workspaceDir` için Docker korumalı alanını hazırlar. Canlı yapılandırma karması istenen bağlamalarla veya politikayla eşleşmeyen sıcak bir kapsayıcıyı
    reddeder. Yalnızca kayıtlı uygulamaları çağıran plugin tarafından sınırlandırılan tam araç adlarını
    iletin; joker karakterli ön ekler araç sahipliğini kanıtlamaz.

  </Accordion>
  <Accordion title="api.runtime.nodes">
    Bağlı nodeları listeleyin ve Gateway tarafından yüklenen plugin kodundan veya plugin CLI komutlarından bir node ana bilgisayarı komutu çağırın. Bir plugin, eşleştirilmiş bir cihazdaki yerel çalışmanın sahibi olduğunda bunu kullanın; örneğin başka bir Mac'teki tarayıcı veya ses köprüsü.

    ```typescript
    const { nodes } = await api.runtime.nodes.list({ connected: true });

    const result = await api.runtime.nodes.invoke({
      nodeId: "mac-studio",
      command: "my-plugin.command",
      params: { action: "start" },
      timeoutMs: 30000,
    });
    ```

    `nodes.list(...)`, her bağlı Node söz konusu Node ajana Plugin veya MCP destekli
    araçlar sunduğunda, o Node'un yayımladığı `nodePluginTools` tanımlayıcılarını
    içerir. Bu tanımlayıcılar canlı bağlantı durumudur: Node bağlantısı kesildiğinde
    Gateway bunları kaldırır ve yerel Plugin/MCP envanteri değiştikten sonra bir Node
    bunları `node.pluginTools.update` ile değiştirebilir.

    Gateway içinde bu çalışma zamanı süreç içindedir. Plugin CLI komutlarında yapılandırılmış Gateway'i RPC üzerinden çağırır; böylece `openclaw googlemeet recover-tab` gibi komutlar eşleştirilmiş Node'ları terminalden inceleyebilir. Node komutları yine normal Gateway Node eşleştirmesinden, komut izin listelerinden, Plugin Node çağırma ilkelerinden ve Node'a yerel komut işleme sürecinden geçer.

    Node üzerinde barındırılan ajan araçları sunan Plugin'ler, varsayılan olarak izin listesine alınması gereken tehlikesiz komutlar için `agentTool.defaultPlatforms` ayarlayabilir. Operatörlerin `gateway.nodes.commands.allow` ile açıkça etkinleştirmesi gerektiğinde bunu belirtmeyin. Tehlikeli Node ana makine komutları, `api.registerNodeInvokePolicy(...)` ile bir Node çağırma ilkesi kaydetmelidir; ilke, komut izin listesi denetimlerinden sonra ve komut Node'a iletilmeden önce Gateway'de çalışır. Böylece doğrudan `node.invoke` çağrıları, Node üzerinde barındırılan Plugin araçları ve üst düzey Plugin araçları aynı yaptırım yolunu paylaşır.

    <Warning>
    İsteğe bağlı `scopes` alanı, çağrı için Gateway operatör kapsamları ister. OpenClaw bunu yalnızca paketlenmiş Plugin'ler ve güvenilir resmî Plugin kurulumları için dikkate alır; diğer Plugin'lerden gelen istekler çağrının yetkisini yükseltmez. Yalnızca güvenilir bir Plugin'in `operator.admin` gibi daha katı bir Gateway kapsamıyla bir Node komutu çağırması gerektiğinde kullanın.
    </Warning>

  </Accordion>
  <Accordion title="api.runtime.tasks">
    Task Flow ve Task Run durumunu mevcut bir OpenClaw oturum anahtarına veya güvenilir araç bağlamına bağlar.

    - `api.runtime.tasks.managedFlows` değişiklik yapabilir: Task Flow'ları oluşturur, ilerletir ve iptal eder.
    - `api.runtime.tasks.flows` ve `api.runtime.tasks.runs`, listeleme ve durum sorgulamaları için salt okunur DTO görünümleridir; her ikisi de `bindSession(...)` / `fromToolContext(...)` ile birlikte `get`, `list`, `findLatest` ve `resolve` alanlarını sunar.

    Task Flow, kalıcı çok adımlı iş akışı durumunu izler. Bir zamanlayıcı değildir:
    gelecekteki uyandırmalar için Cron veya `api.session.workflow.scheduleSessionTurn(...)` kullanın;
    ardından bu çalışma akış durumu, alt görevler, beklemeler veya iptal gerektirdiğinde
    zamanlanmış turdan `managedFlows` kullanın.

    ```typescript
    const taskFlow = api.runtime.tasks.managedFlows.fromToolContext(ctx);

    const created = taskFlow.createManaged({
      controllerId: "my-plugin/review-batch",
      goal: "Yeni pull request'leri incele",
    });

    const child = taskFlow.runTask({
      flowId: created.flowId,
      runtime: "acp",
      childSessionKey: "agent:main:subagent:reviewer",
      task: "PR #123'ü incele",
      status: "running",
      startedAt: Date.now(),
    });

    const waiting = taskFlow.setWaiting({
      flowId: created.flowId,
      expectedRevision: created.revision,
      currentStep: "await-human-reply",
      waitJson: { kind: "reply", channel: "telegram" },
    });
    ```

    Kendi bağlama katmanınızdan zaten güvenilir bir OpenClaw oturum anahtarınız olduğunda `bindSession({ sessionKey, requesterOrigin })` kullanın. Ham kullanıcı girdisinden bağlama yapmayın.

  </Accordion>
  <Accordion title="api.runtime.tts">
    Metinden konuşmaya sentezi.

    ```typescript
    // Standart TTS
    const clip = await api.runtime.tts.textToSpeech({
      text: "OpenClaw'dan merhaba",
      cfg: api.config,
    });

    // Telefon iletişimi için optimize edilmiş TTS
    const telephonyClip = await api.runtime.tts.textToSpeechTelephony({
      text: "OpenClaw'dan merhaba",
      cfg: api.config,
    });

    // Kullanılabilir sesleri listele
    const voices = await api.runtime.tts.listVoices({
      provider: "elevenlabs",
      cfg: api.config,
    });
    ```

    Temel `tts` yapılandırmasını ve sağlayıcı seçimini kullanır. PCM ses arabelleği + örnekleme hızı döndürür. Akışlı sentez için `textToSpeechStream` de kullanılabilir.

  </Accordion>
  <Accordion title="api.runtime.mediaUnderstanding">
    Görüntü, ses ve video analizi.

    ```typescript
    // Bir görüntüyü açıkla
    const image = await api.runtime.mediaUnderstanding.describeImageFile({
      filePath: "/tmp/inbound-photo.jpg",
      cfg: api.config,
      agentDir: "/tmp/agent",
    });

    // Sesi yazıya dök
    const { text } = await api.runtime.mediaUnderstanding.transcribeAudioFile({
      filePath: "/tmp/inbound-audio.ogg",
      cfg: api.config,
      mime: "audio/ogg", // isteğe bağlı, MIME çıkarılamadığında kullanılır
    });

    // Bir videoyu açıkla
    const video = await api.runtime.mediaUnderstanding.describeVideoFile({
      filePath: "/tmp/inbound-video.mp4",
      cfg: api.config,
    });

    // Genel dosya analizi
    const result = await api.runtime.mediaUnderstanding.runFile({
      filePath: "/tmp/inbound-file.pdf",
      cfg: api.config,
    });

    // Belirli bir sağlayıcı/model aracılığıyla yapılandırılmış görüntü çıkarımı.
    // En az bir görüntü ekleyin; metin girdileri ek bağlam sağlar.
    const evidence = await api.runtime.mediaUnderstanding.extractStructuredWithModel({
      provider: "codex",
      model: "gpt-5.6-sol",
      input: [
        {
          type: "image",
          buffer: receiptImageBuffer,
          fileName: "receipt.png",
          mime: "image/png",
        },
        { type: "text", text: "El yazısı notlar yerine basılı toplamı tercih et." },
      ],
      instructions: "Satıcıyı, toplamı ve aranabilir etiketleri çıkar.",
      schemaName: "receipt.evidence",
      jsonSchema: {
        type: "object",
        properties: {
          vendor: { type: "string" },
          total: { type: "number" },
          tags: { type: "array", items: { type: "string" } },
        },
        required: ["vendor", "total"],
      },
      cfg: api.config,
    });
    ```

    Hiçbir çıktı üretilmediğinde (ör. atlanan girdi) `{ text: undefined }` döndürür.

    `describeImageFileWithModel(...)`, `describeImageFile(...)` tarafından kullanılan varsayılan etkin model çözümlemesini atlayarak önceden bilinen bir görüntüyü belirli bir sağlayıcı/model aracılığıyla açıklar.

  </Accordion>
  <Accordion title="api.runtime.imageGeneration">
    Görüntü oluşturma.

    ```typescript
    const result = await api.runtime.imageGeneration.generate({
      prompt: "Gün batımını resmeden bir robot",
      cfg: api.config,
    });

    const providers = api.runtime.imageGeneration.listProviders({ cfg: api.config });
    ```

  </Accordion>
  <Accordion title="api.runtime.videoGeneration">
    Görüntü oluşturma biçimini yansıtan video oluşturma.

    ```typescript
    const result = await api.runtime.videoGeneration.generate({
      prompt: "Gün doğumunda bir kıyı şeridi üzerinde uçan drone çekimi",
      cfg: api.config,
    });

    const providers = api.runtime.videoGeneration.listProviders({ cfg: api.config });
    ```

  </Accordion>
  <Accordion title="api.runtime.musicGeneration">
    Görüntü oluşturma biçimini yansıtan müzik oluşturma.

    ```typescript
    const result = await api.runtime.musicGeneration.generate({
      prompt: "Bir kodlama oturumu için hareketli bir lo-fi parçası",
      cfg: api.config,
    });

    const providers = api.runtime.musicGeneration.listProviders({ cfg: api.config });
    ```

  </Accordion>
  <Accordion title="api.runtime.webSearch">
    Web araması.

    ```typescript
    const providers = api.runtime.webSearch.listProviders({ config: api.config });

    const result = await api.runtime.webSearch.search({
      config: api.config,
      args: { query: "OpenClaw Plugin SDK", count: 5 },
    });
    ```

  </Accordion>
  <Accordion title="api.runtime.media">
    Düşük düzeyli medya yardımcı araçları.

    ```typescript
    const webMedia = await api.runtime.media.loadWebMedia(url);
    const mime = await api.runtime.media.detectMime(buffer);
    const kind = api.runtime.media.mediaKindFromMime("image/jpeg"); // "image"
    const isVoice = api.runtime.media.isVoiceCompatibleAudio(filePath);
    const metadata = await api.runtime.media.getImageMetadata(filePath);
    const resized = await api.runtime.media.resizeToJpeg(buffer, { maxWidth: 800 });
    const terminalQr = await api.runtime.media.renderQrTerminal("https://openclaw.ai");
    const pngQr = await api.runtime.media.renderQrPngBase64("https://openclaw.ai", {
      scale: 6, // 1-12
      marginModules: 4, // 0-16
    });
    const pngQrDataUrl = await api.runtime.media.renderQrPngDataUrl("https://openclaw.ai");
    const tmpRoot = resolvePreferredOpenClawTmpDir();
    const pngQrFile = await api.runtime.media.writeQrPngTempFile("https://openclaw.ai", {
      tmpRoot,
      dirPrefix: "my-plugin-qr-",
      fileName: "qr.png",
    });
    ```

  </Accordion>
  <Accordion title="api.runtime.config">
    Geçerli çalışma zamanı yapılandırma anlık görüntüsü ve işlemsel yapılandırma yazımları. Etkin çağrı yoluna
    zaten aktarılmış yapılandırmayı tercih edin; `current()` öğesini yalnızca işleyicinin
    süreç anlık görüntüsüne doğrudan ihtiyaç duyduğu durumlarda kullanın.

    ```typescript
    const cfg = api.runtime.config.current();
    await api.runtime.config.mutateConfigFile({
      afterWrite: { mode: "auto" },
      mutate(draft) {
        draft.plugins ??= {};
      },
    });
    ```

    `mutateConfigFile(...)` ve `replaceConfigFile(...)`, örneğin `{ mode: "restart", requiresRestart: true, reason }`
    gibi bir `followUp` değeri döndürür;
    bu değer, yeniden başlatma denetimini Gateway'den almadan yazarın amacını
    kaydeder.

  </Accordion>
  <Accordion title="api.runtime.system">
    Sistem düzeyinde yardımcı araçlar.

    ```typescript
    await api.runtime.system.enqueueSystemEvent(event);
    api.runtime.system.requestHeartbeat({
      source: "other",
      intent: "event",
      reason: "plugin-event",
    });
    api.runtime.system.requestHeartbeatNow({ reason: "plugin-event" }); // Kullanımdan kaldırılmış uyumluluk diğer adı.
    const heartbeatResult = await api.runtime.system.runHeartbeatOnce({
      reason: "plugin-triggered-check",
    });
    const output = await api.runtime.system.runCommandWithTimeout(cmd, args, opts);
    const hint = api.runtime.system.formatNativeDependencyHint(pkg);
    ```

    `runHeartbeatOnce(...)`, normal birleştirme zamanlayıcısını atlayarak tek bir Heartbeat döngüsünü hemen çalıştırır. Varsayılan `target: "none"` engellemesi yerine son etkin kanala teslimatı zorlamak için `{ heartbeat: { target: "last" } }` iletin.

    `runCommandWithTimeout(...)`; yakalanan `stdout` ve `stderr`, isteğe bağlı
    kesme sayıları, `code`, `signal`, `killed`, `termination` ve
    `noOutputTimedOut` değerlerini döndürür. Zaman aşımı ve çıktı yokluğu zaman aşımı sonuçları,
    alt süreç sıfır olmayan bir çıkış kodu sağlamadığında `code: 124`
    bildirir. Zaman aşımı dışındaki sinyal çıkışları yine de `code: null` döndürebilir;
    bu nedenle zaman aşımı nedenlerini ayırt etmek için `termination` ve
    `noOutputTimedOut` kullanın.

  </Accordion>
  <Accordion title="api.runtime.events">
    Olay abonelikleri.

    ```typescript
    api.runtime.events.onAgentEvent((event) => {
      /* ... */
    });
    api.runtime.events.onSessionTranscriptUpdate((update) => {
      /* ... */
    });
    ```

  </Accordion>
  <Accordion title="api.runtime.logging">
    Günlük kaydı.

    ```typescript
    const verbose = api.runtime.logging.shouldLogVerbose();
    const childLogger = api.runtime.logging.getChildLogger({ plugin: "my-plugin" }, { level: "debug" });
    ```

  </Accordion>
  <Accordion title="api.runtime.modelAuth">
    Model ve sağlayıcı kimlik doğrulaması çözümlemesi.

    ```typescript
    const auth = await api.runtime.modelAuth.getApiKeyForModel({ model, cfg });

    // Sağlayıcı çalışma zamanı değişimleri (ör. OAuth yenileme) dâhil, isteğe hazır kimlik doğrulama
    const runtimeAuth = await api.runtime.modelAuth.getRuntimeAuthForModel({ model, cfg });

    const providerAuth = await api.runtime.modelAuth.resolveApiKeyForProvider({
      provider: "openai",
      cfg,
    });
    ```

  </Accordion>
  <Accordion title="api.runtime.state">
    Durum dizini çözümleme ve SQLite destekli anahtarlı depolama.

    ```typescript
    const stateDir = api.runtime.state.resolveStateDir(process.env);
    const store = api.runtime.state.openKeyedStore<MyRecord>({
      namespace: "my-feature",
      maxEntries: 200,
      defaultTtlMs: 15 * 60_000,
    });

    await store.register("key-1", { value: "hello" });
    const claimed = await store.registerIfAbsent("dedupe-key", { value: "first" });
    const value = await store.lookup("key-1");
    await store.deleteIf?.("key-1", (current) => current.value === "hello");
    await store.consume("key-1");
    await store.clear();

    const blobs = api.runtime.state.openBlobStore<MyBlobMetadata>({
      namespace: "rendered-artifacts",
      maxEntries: 100,
      maxBytesPerEntry: 4 * 1024 * 1024,
      maxBytesPerNamespace: 64 * 1024 * 1024,
      defaultTtlMs: 15 * 60_000,
    });
    await blobs.register(
      "artifact-1",
      new TextEncoder().encode("binary or text payload"),
      { contentType: "text/plain" },
    );
    const blob = await blobs.lookup("artifact-1");

    await api.runtime.state.withLease(
      {
        namespace: "my-feature",
        key: "writer",
        database: { scope: "agent", agentId },
        leaseMs: 5 * 60_000,
        waitMs: 30_000,
      },
      async ({ signal, assertOwned }) => {
        await runExternalWriter({ signal });
        assertOwned();
      },
    );
    ```

    Anahtarlı depolar yeniden başlatmalardan etkilenmez ve çalışma zamanına bağlı plugin kimliğine göre yalıtılır. Atomik tekilleştirme talepleri için `registerIfAbsent(...)` kullanın: anahtar eksikse veya süresi dolmuşsa ve kaydedildiyse `true`; değerinin, oluşturulma zamanının veya TTL'sinin üzerine yazılmadan etkin bir değer zaten varsa `false` döndürür. Temizliğin yalnızca daha önce gözlemlenen değeri kaldırması gerektiğinde `deleteIf(...)` kullanın; eşzamanlı koşulu ve silme işlemi tek bir SQLite işlemi içinde yürütülür. Sınırlar: ad alanı başına `maxEntries`, plugin başına 50.000 etkin satır, 64KB altındaki JSON değerleri ve isteğe bağlı TTL süre sonu. Varsayılan olarak, satır sınırlarından herhangi birinde yapılan bir yazma işlemi, yazılmakta olan ad alanındaki en eski etkin satırları çıkarır; bu yazma için kardeş ad alanları çıkarılmaz ve ad alanı yeterli sayıda satırı boşaltamazsa yazma işlemi yine başarısız olur. Hiçbir zaman çıkarılmaması gereken kalıcı sahiplik kayıtları için `overflowPolicy: "reject-new"` ayarlayın: yeni anahtarlar her iki sınırda da başarısız olurken mevcut anahtarlar güncellenebilir durumda kalır.

    `openSyncKeyedStore<T>(...)`, bekleyemeyen çağıranlar için eşzamanlı yöntemlerle aynı depo biçimini döndürür (`register`, `registerIfAbsent`, `deleteIf`, `lookup`, `consume`, `clear` yöntemlerinin tümü promise yerine değerleri doğrudan döndürür).

    `openBlobStore<TMetadata>(...)`, sınırlı ikili yükleri base64 veya dosya yan kayıtları olmadan paylaşılan SQLite'ta depolar. Girdi başına, ad alanı başına bayt ve satır sınırları gerektirir; API sınırında bayt dizilerini kopyalar ve her BLOB'u yüklemeden meta verileri listeler. `register(...)`, süresi dolmuş anahtarlar dâhil olmak üzere açık bir upsert işlemidir. `registerIfAbsent(...)`, çakışmaya dayanıklı oluşturma sağlar: süresi dolmuş bir anahtar, sahibi `deleteExpiredKey(key)` veya `deleteExpired()` ile anahtarı talep edene kadar dolu kalır; böylece SQLite işlemi tamamlandıktan sonra ilgili adlandırılmış yapıtları kaldırmak için gereken meta veriler korunur. TTL içeren her satır geçicidir ve süresi dolmadan önce bile yedekleme/geri yüklemenin dışında tutulur; kalıcı ve geri yüklenebilir durum için TTL'yi atlayın. Ana makine sigortaları her BLOB'u 100 MiB, her plugini fiziksel olarak depolanan 512 MiB BLOB ve her plugini, sahibi tarafından temizlenmeyi bekleyen süresi dolmuş satırlar dâhil olmak üzere fiziksel olarak depolanan 50.000 satırla sınırlar. Harici somutlaştırmaların değiştirme veya çıkarma nedeniyle sessizce sahipsiz kalmaması gerektiğinde `registerIfAbsent(...)` ile birlikte `overflowPolicy: "reject-new"` kullanın.

    `openChannelIngressQueue<TPayload>(...)`, yeniden başlatmalar arasında en az bir kez işlenmesi gereken gelen olayları arabelleğe almak için çağıran plugin kapsamında kalıcı bir giriş kuyruğu açar. Eski talep kurtarma işlemi `shouldRecover` kullandığında, bozuk talep edilmiş yüklerin karantinaya alınması gerekiyorsa `shouldRecoverCorrupt` değerini de sağlayın: yükten bağımsız talep kimliği, kuyruk satırı mezar taşıyla işaretlemeden önce pluginin etkin sahip ve şerit politikasını korumasına olanak tanır.

    `withLease(...)`, ortak çalışmaya dayalı plugin işlerini OpenClaw süreçleri arasında serileştirir. Tek bir genel sahip için `database: { scope: "shared" }`, aracı başına bağımsız sahiplik için `{ scope: "agent", agentId }` seçin. Geri çağırmanın `AbortSignal` değerini başarısız olabilecek her işleme iletin. `assertOwned()`, başka bir önemli adıma başlamadan önce belirli bir andaki denetim noktasıdır; ana makine geri çağırmadan sonra da sahipliği doğrular. Kiralama kaybı veya çağıranın iptali sinyali sonlandırır. Edinme beklemeleri ve heartbeat'ler kısa eşzamanlı SQLite işlemlerinin dışında gerçekleşir; pluginler hiçbir zaman veritabanı yollarını veya tanıtıcılarını almaz. Bu, ortak çalışmaya dayalı iptaldir; çitleme belirteci veya çitlenmemiş harici yazmalar için yetkilendirme değildir.

    `openChannelIngressDrain(...)`, bu kuyruk üzerinde kanaldan bağımsız çekirdek worker'ı açar (veya sağlanmamışsa bir kuyruk oluşturur). Boşaltma işlemi; eski talep kurtarmanın, şerit başına talep serileştirmenin, benimsemede tamamlama veya gönderim dönüşünde tamamlamanın, yeniden deneme/ölü mektup yerleşiminin, isteğe bağlı benimseme öncesi geçersiz kılmanın ve talep→benimseme duraklama zaman aşımının sahibidir. Talep sahipliğini `turnAdoptionLifecycle` ile yanıt oluşturmaya bağlayın (`plugin-sdk/channel-outbound` üzerinden `bindIngressLifecycleToReplyOptions` aracılığıyla). Kanal pluginleri kabul tarafı kuyruğa ekleme, şerit türetme, yeniden denenemez sınıflandırma ve tüm geçersiz kılma yetkilendirme politikalarını elinde tutar.

    <Warning>
    `openBlobStore`, `openKeyedStore`, `openSyncKeyedStore`, `withLease`, `openChannelIngressQueue` ve `openChannelIngressDrain` bu sürümde yalnızca paketlenmiş pluginler ve güvenilir resmî plugin kurulumları tarafından kullanılabilir.
    </Warning>

  </Accordion>
  <Accordion title="api.runtime.channel">
    Kanala özgü çalışma zamanı yardımcıları (bir kanal plugini yüklendiğinde kullanılabilir). İlgili alana göre gruplandırılmıştır:

    | Grup | Amaç |
    | --- | --- |
    | `text` | Parçalama (`chunkText`, `chunkMarkdownText`, `resolveChunkMode`), denetim komutu algılama, Markdown tablosu dönüştürme. |
    | `reply` | Arabelleğe alınmış blok yanıtı gönderimi, zarf biçimlendirme, geçerli mesaj/insan gecikmesi yapılandırması çözümleme. |
    | `routing` | `buildAgentSessionKey`, `resolveAgentRoute`. |
    | `pairing` | `buildPairingReply`, izin listesi okuma/kaldırma, eşleştirme isteği upsert işlemleri ve istekten türetilen onay girdileri. |
    | `media` | Uzak medya indirme/kaydetme (aşağıya bakın). |
    | `activity` | Son kanal etkinliğini kaydetme/okuma. |
    | `session` | Gelen olaylardan oturum meta verileri, son rota güncellemeleri. |
    | `mentions` | Bahsetme politikası yardımcıları (aşağıya bakın). |
    | `reactions` | Devam eden işlem göstergeleri için onay tepkisi tanıtıcıları. |
    | `groups` | Grup politikası ve bahsetme gereksinimi çözümleme. |
    | `debounce` | Gelen mesajların yinelenmesini önleme. |
    | `commands` | Komut yetkilendirme ve metin komutu geçitleme. |
    | `outbound` | Bir kanalın giden bağdaştırıcısını yükleme. |
    | `inbound` | Gelen olay bağlamını oluşturma ve paylaşılan gelen olay/yanıt çekirdeğini çalıştırma. |
    | `threadBindings` | Bağlı oturum iş parçacıkları için boşta kalma zaman aşımını/azami yaşı ayarlama. |
    | `runtimeContexts` | Süreç yerelinde kanal/hesap/yetenek başına bağlamı kaydetme, okuma ve izleme. |

    `api.runtime.channel.media`, kanal medyası indirmeleri ve depolaması için tercih edilen yüzeydir:

    ```typescript
    const saved = await api.runtime.channel.media.saveRemoteMedia({
      url,
      subdir: "inbound",
      maxBytes,
      filePathHint: fileName,
    });
    ```

    Uzak bir URL'nin OpenClaw medyasına dönüşmesi gerektiğinde `saveRemoteMedia(...)` kullanın. Plugin, pluginin sahip olduğu kimlik doğrulama, yönlendirme veya izin listesi işleme mekanizmasıyla bir `Response` zaten getirmişse `saveResponseMedia(...)` kullanın. `readRemoteMediaBuffer(...)` öğesini yalnızca pluginin inceleme, dönüştürme, şifre çözme veya yeniden yükleme için ham baytlara ihtiyacı olduğunda kullanın. `fetchRemoteMedia(...)`, `readRemoteMediaBuffer(...)` için kullanımdan kaldırılmış bir uyumluluk diğer adı olarak kalır.

    `api.runtime.channel.mentions`, çalışma zamanı ekleme kullanan paketlenmiş kanal pluginleri için paylaşılan gelen bahsetme politikası yüzeyidir:

    ```typescript
    const mentionMatch = api.runtime.channel.mentions.matchesMentionWithExplicit(text, {
      mentionRegexes,
      mentionPatterns,
    });

    const decision = api.runtime.channel.mentions.resolveInboundMentionDecision({
      facts: {
        canDetectMention: true,
        wasMentioned: mentionMatch.matched,
        implicitMentionKinds: api.runtime.channel.mentions.implicitMentionKindWhen(
          "reply_to_bot",
          isReplyToBot,
        ),
      },
      policy: {
        isGroup,
        requireMention,
        allowTextCommands,
        hasControlCommand,
        commandAuthorized,
      },
    });
    ```

    Kullanılabilir bahsetme yardımcıları:

    - `buildMentionRegexes`
    - `matchesMentionPatterns`
    - `matchesMentionWithExplicit`
    - `implicitMentionKindWhen`
    - `resolveInboundMentionDecision`

    Bahsetme kararları için normalleştirilmiş `{ facts, policy }` yolunu kullanın.

    `reply`, `session` ve `inbound` altındaki çeşitli alanlar, geçerli kanal dönüşü çekirdeğine veya kanal giden bağdaştırıcılarına işaret eden alan başına `@deprecated` notları taşır; üzerinde yeni kod oluşturmadan önce ilgili yardımcının satır içi JSDoc belgesini kontrol edin.

  </Accordion>
</AccordionGroup>

## Çalışma zamanı referanslarını depolama

Çalışma zamanı referansını `register` geri çağırması dışında kullanmak üzere depolamak için `createPluginRuntimeStore` kullanın:

<Steps>
  <Step title="Depoyu oluşturun">
    ```typescript
    import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
    import type { PluginRuntime } from "openclaw/plugin-sdk/runtime-store";

    const store = createPluginRuntimeStore<PluginRuntime>({
      pluginId: "my-plugin",
      errorMessage: "my-plugin runtime not initialized",
    });
    ```

  </Step>
  <Step title="Giriş noktasına bağlayın">
    ```typescript
    export default defineChannelPluginEntry({
      id: "my-plugin",
      name: "My Plugin",
      description: "Example",
      plugin: myPlugin,
      setRuntime: store.setRuntime,
    });
    ```
  </Step>
  <Step title="Diğer dosyalardan erişin">
    ```typescript
    export function getRuntime() {
      return store.getRuntime(); // başlatılmadıysa hata oluşturur
    }

    export function tryGetRuntime() {
      return store.tryGetRuntime(); // başlatılmadıysa null döndürür
    }
    ```

  </Step>
</Steps>

<Note>
Çalışma zamanı deposu kimliği için `pluginId` tercih edin. Alt düzey `key` biçimi, bir pluginin bilinçli olarak birden fazla çalışma zamanı yuvasına ihtiyaç duyduğu nadir durumlar içindir.
</Note>

## Diğer üst düzey `api` alanları

`api.runtime` dışında API nesnesi şunları da sağlar:

<ParamField path="api.id" type="string">
  Plugin kimliği.
</ParamField>
<ParamField path="api.name" type="string">
  Plugin görünen adı.
</ParamField>
<ParamField path="api.config" type="OpenClawConfig">
  Geçerli yapılandırma anlık görüntüsü (varsa etkin bellek içi çalışma zamanı anlık görüntüsü).
</ParamField>
<ParamField path="api.pluginConfig" type="Record<string, unknown>">
  `plugins.entries.<id>.config` kaynağından Plugin'a özgü yapılandırma.
</ParamField>
<ParamField path="api.logger" type="PluginLogger">
  Kapsamlı günlükçü (`debug`, `info`, `warn`, `error`).
</ParamField>
<ParamField path="api.registrationMode" type="PluginRegistrationMode">
  Geçerli yükleme modu: `"full"` (canlı etkinleştirme), `"discovery"` / `"tool-discovery"` (salt okunur yetenek keşfi), `"setup-only"` (hafif kurulum giriş noktası), `"setup-runtime"` (çalışma zamanı kanal girişine de ihtiyaç duyan kurulum akışı) veya `"cli-metadata"` (CLI komut meta verilerinin toplanması).
</ParamField>
<ParamField path="api.resolvePath(input)" type="(string) => string">
  Plugin köküne göre göreli bir yolu çözümleyin.
</ParamField>

## İlgili

- [Plugin iç yapısı](/tr/plugins/architecture) — yetenek modeli ve kayıt defteri
- [SDK giriş noktaları](/tr/plugins/sdk-entrypoints) — `definePluginEntry` seçenekleri
- [SDK'ya genel bakış](/tr/plugins/sdk-overview) — alt yol başvurusu
